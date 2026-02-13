# ARC Raiders Discord Bot (JDA) – Spec (.md) für Sonnet
**Stack:** Java 17+, JDA, `java.net.http.HttpClient`, Gson, Jsoup (optional Playwright-Fallback), keine SQL-DB.

---

## Ziele

### 1) Weekly Trials (Scrape)
- Quelle: `https://metaforge.app/arc-raiders/weekly-trials`
- Zeitplan: **jeden Montag 09:00 (Europe/Berlin)**
- Verhalten:
  - Bot postet **eine neue** Trials-Nachricht in den konfigurierten Trials-Channel
  - **Vorherige Trials-Bot-Nachricht wird gelöscht**, bevor die neue gesendet wird (siehe Permissions)
  - Optional: Wenn Trials-Channel ein **Announcement Channel** ist, kann die Nachricht **gepublished/crossposted** werden, damit andere Server dem Channel folgen können

### 2) Events Overview (API, persistente Nachricht)
- Quelle: MetaForge API
- In einem konfigurierten Events-Channel existiert **genau eine Bot-Nachricht**, die **immer editiert** wird
- Die Nachricht enthält **5 Embeds (pro Event eins)** und zeigt nur Slots im Fenster **15:00–02:00 (Europe/Berlin)**

---

## Commands (Slash)

### `/trials channel:<#channel>`
- Setzt den Channel für Weekly Trials
- Speichert `weeklyChannelId` in `config.json`
- Optional: kann nach Setzen direkt einen “Testpost” machen (nicht erforderlich)

### `/events channel:<#channel>`
- Setzt den Channel für Events Overview
- Speichert `eventsChannelId`
- Erstellt die persistente Overview-Nachricht (falls nicht vorhanden) und speichert `eventsMessageId`

### `/refresh`
- Forciert ein sofortiges Refresh der Events Overview Message (ohne TTL)
- Antwort ephemeral: Erfolg/Fehler

> Hinweis: Es gibt **kein** Setup-Command mehr – die Channel-Settings passieren über `/trials` und `/events`.

---

## Datenquellen

### A) Events API (MetaForge)
- Base URL: `https://metaforge.app/api/arc-raiders`
- Endpoint: `GET /api/arc-raiders/events-schedule`
- Response:
```json
{
  "data": [
    { "name": "Matriarch", "map": "Dam", "icon": "https://...", "startTime": 1770890400000, "endTime": 1770894000000 }
  ],
  "cachedAt": 1770892101739
}
```

#### Relevante Events (Whitelist)
Nur diese Events anzeigen:
- `Night Raid`
- `Electromagnetic Storm`
- `Hidden Bunker`
- `Locked Gate`
- `Cold Snap`

#### Zeitfenster Filter (15:00–02:00 Europe/Berlin)
- `startTime/endTime` sind **epoch ms (UTC)**
- Konvertiere konsequent nach `Europe/Berlin` (DST beachten)
- Definiere Fenster:
  - `windowStart = heute 15:00 (Berlin)`
  - `windowEnd   = morgen 02:00 (Berlin)`
- Zeige Event-Slots, die das Fenster **überlappen**:
  - `eventStart < windowEnd && eventEnd > windowStart`

---

### B) Weekly Trials (Scrape)
- URL: `https://metaforge.app/arc-raiders/weekly-trials`
- Scrape-Strategie:
  1) **Jsoup Fast Path:** HTML laden + Trials aus DOM oder aus JSON in `<script>` extrahieren
  2) **Fallback (optional):** Wenn keine Trials gefunden werden (client-rendered), Playwright laden und nach Rendern extrahieren

> Trials-Format ist nicht offiziell dokumentiert; Scraper muss robust sein (tolerant gegen DOM Änderungen).

---

## Discord Output

### A) Events Overview: 1 Message, 5 Embeds (immer editieren)
- Genau **eine** Bot-Message im `eventsChannelId`
- Diese Message hat **5 Embeds** in fester Reihenfolge:
  1. Night Raid
  2. Electromagnetic Storm
  3. Hidden Bunker
  4. Locked Gate
  5. Cold Snap

**Embed-Layout pro Event**
- `title`: Event-Name
- `thumbnail`: `icon` (falls vorhanden)
- `description`: `Termine 15:00–02:00 (Europe/Berlin)`
- Inhalt: Termine **inkl. Map**, z.B. gruppiert nach Map:
  - `Dam: 18:00–19:00, 21:00–22:00`
  - `Spaceport: 19:00–20:00`
- `footer`: `Last updated: <Berlin datetime> | source cachedAt: <cachedAt>`

Wenn keine Termine im Zeitfenster:
- Embed bleibt bestehen:
  - `description`: `Keine Termine im Zeitfenster 15:00–02:00.`

---

### B) Weekly Trials: neue Nachricht, vorherige löschen
- Bot speichert `weeklyLastMessageId` (pro Guild) in `config.json`
- Montags:
  1) wenn `weeklyLastMessageId` existiert: **löschen**
  2) neue Trials-Message senden
  3) neue Message-ID speichern als `weeklyLastMessageId`
  4) optional: Wenn Channel ein Announcement Channel ist, **crosspost/publish** (siehe Permissions)

---

## Caching (ohne SQL) – Pflicht

### Ziele
- API & Scrape Calls minimieren
- Discord Edits minimieren (Rate Limits)

### Mechanik
- 2-stufig:
  1) In-Memory Cache
  2) File Cache `cache.json` (überlebt Neustarts)
- TTL:
  - Events Schedule: **10 Minuten**
  - Weekly Trials Scrape: **12–24 Stunden** (konfigurierbar)
- Change-Detection:
  - Hash über **normalisierte** Daten (sortiert, whitespace-trim, ohne “Last updated”)
  - Events: wenn Hash gleich → **kein editMessageEmbeds()**
  - Trials: TTL + optional Monday-force (nur Montag 09:00 einmal forcieren)

### Fallback
- Quelle down:
  - Nutze letzten Cache auch wenn expired
  - Footer Hinweis: `Source unavailable – showing last known data`

---

## Scheduling

### A) Events Refresh Job
- Alle **10 Minuten**
- Flow:
  1) `getEventsScheduleCached(TTL=10m)`
  2) whitelist + timeWindow(15:00–02:00 Berlin)
  3) build 5 embeds
  4) nur editieren wenn Hash changed

### B) Weekly Trials Job
- **Montag 09:00 Europe/Berlin**
- Flow:
  1) `getWeeklyTrialsCached(TTL=12–24h; optional Monday-force)`
  2) wenn `weeklyLastMessageId` vorhanden → löschen
  3) neue Trials message senden
  4) optional publish bei Announcement Channel

---

## Persistenz (JSON Files)

### `config.json` (Beispiel)
```json
{
  "guilds": {
    "GUILD_ID": {
      "weeklyChannelId": "CHANNEL_ID",
      "eventsChannelId": "CHANNEL_ID",
      "eventsMessageId": "MESSAGE_ID",
      "weeklyLastMessageId": "MESSAGE_ID"
    }
  }
}
```

### `cache.json` (Beispiel)
```json
{
  "eventsSchedule": {
    "fetchedAtMs": 0,
    "expiresAtMs": 0,
    "contentHash": "abc",
    "payload": { "data": [], "cachedAt": 0 }
  },
  "weeklyTrials": {
    "fetchedAtMs": 0,
    "expiresAtMs": 0,
    "contentHash": "def",
    "payload": { "trials": [], "fetchedAtMs": 0 }
  }
}
```

**File I/O**
- Atomic write: temp file → move/replace
- Synchronisiertes Schreiben (Jobs parallel)

---

## HttpClient Regeln
- Singleton `HttpClient`
- Request timeout ~10s, connect timeout ~5–10s
- Retries max 2 bei IO/5xx/429
- 429:
  - `Retry-After` respektieren, sonst Backoff (z.B. 500ms, 1500ms)
- User-Agent setzen: `ARC-Raiders-DiscordBot/1.0`

---

## Permissions (Discord) – wichtig

### Für Events (editieren)
- `Send Messages`
- `Embed Links`
- `Read Message History` (um Message zu laden/validieren)

### Für Weekly Trials “delete previous message”
- **Delete Message erfordert `Manage Messages`.** citeturn0search6  
  → Wenn du **wirklich** vorherige Message löschen willst, muss der Bot **Manage Messages** im Trials-Channel haben.  
  Alternative (falls du Manage Messages nicht geben willst): statt delete → **edit** der vorherigen Trials-Message.

### Announcement Channel / Publish / Crosspost
- Discord Announcement Channels können gepostet und “published” werden, wenn die passenden Rechte gesetzt sind. citeturn0search11turn0search17
  - Empfehlung: Wenn `channel.isNews()` (JDA) → nach dem Senden optional `message.crosspost()` aufrufen (permission-gated; bei fehlenden Rechten skip + log).

---

## Projektstruktur (Empfehlung)
- `api/` (HttpClient wrapper, DTOs)
- `cache/` (CacheService, file persistence, hashing)
- `scrape/` (WeeklyTrialsScraper)
- `discord/` (EmbedBuilder, Message edit/post/delete/crosspost)
- `jobs/` (EventsRefreshJob, WeeklyTrialsJob)
- `commands/` (`/trials`, `/events`, `/refresh`)

---

## TODO (Trials Scrape)
- DOM/Selectoren finalisieren:
  - Sobald klar ist, wie Trials im HTML strukturiert sind:
    - robuste Jsoup Selectoren implementieren
    - Playwright nur als Fallback, wenn serverseitig keine Trials-Infos vorhanden sind
