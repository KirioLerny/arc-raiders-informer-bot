# Plan: ARC Raiders Discord Bot (Weekly Trials + Events Embed)

## Ziel
Der Bot soll:
1. **Jeden Montag um 09:00 (Europe/Berlin)** die aktuellen **Weekly Trials** in einen definierten Channel posten.
2. Auf einer **“Trials-Übersicht” Page (z.B. #trials-info)** **ein einziges Embed** verwalten, das alle für Trials wichtigen **Events/Schedules** anzeigt.
    - Dieses Embed wird **immer bearbeitet (edited)** statt jedes Mal neu gesendet.

---

## Funktionsumfang

### 1) Weekly Trials Push (Montag 09:00)
- Automatischer Job, der:
    - Trials-Daten abruft
    - ein formatiertes Embed erstellt
    - es in einem konfigurierten Channel postet (oder optional: denselben Post editiert, siehe Erweiterungen)

**Output:** 1 Nachricht pro Woche (neu) mit den Trials.

---

### 2) Persistentes “Trials Events” Embed (eine Nachricht, die immer editiert wird)
- In einem festen Channel gibt es **eine** Bot-Nachricht (Embed).
- Der Bot aktualisiert diese Nachricht regelmäßig (z.B. alle 5–15 Minuten) oder bei Datenänderung.
- Inhalt: “Alle für Trials wichtigen Events” (Rotationen, Timers, Special Events, etc.).

**Output:** Immer nur **eine** Nachricht, die sich aktualisiert.

---

## Komponenten / Architektur

### A) Daten-Layer (Fetcher)
**Aufgabe:** Quelle(n) abfragen und in ein einheitliches Format bringen.

- `fetchWeeklyTrials()` → liefert Liste der Trials
- `fetchTrialRelevantEvents()` → liefert Events/Rotations/Schedule

> Hinweis: Wenn es (noch) keine offizielle API gibt, kannst du zunächst:
> - eine eigene JSON-Datei/Endpoint pflegen
> - oder eine Community-Quelle scrapen (mit Caching & Fehlerhandling)
> - später austauschbar auf “offizielle” Quellen wechseln

---

### B) Formatter (Embeds)
**Aufgabe:** Aus Daten schöne Discord-Embeds bauen.

- `buildWeeklyTrialsEmbed(trials)`
- `buildEventsOverviewEmbed(events)`

Embed-Design Vorschlag:
- Titel + Zeitstempel “Last updated: …”
- Felder:
    - “Active Events”
    - “Next Up”
    - “Ends In / Starts In”
- Footer: “ARC Raiders – Unofficial Bot”

---

### C) Scheduler (Cron / Jobs)
**Aufgabe:** Jobs zeitgesteuert ausführen.

1. **Weekly Trials Job**
    - Cron: `0 9 * * 1` (Montag 09:00)
    - Timezone: `Europe/Berlin`

2. **Events Overview Refresh Job**
    - Intervall: z.B. alle 10 Minuten
    - Oder: alle 5 Minuten + “nur editieren wenn Inhalt sich geändert hat”

---

### D) Storage (Config + Message Tracking)
Damit das “eine Embed” immer editiert werden kann, musst du die Message-ID speichern.

Minimal nötig:
- `guild_id`
- `events_channel_id`
- `events_message_id` (die persistente Nachricht)
- `weekly_channel_id`

Speicheroptionen:
- SQLite (empfohlen für klein/mittel)
- JSON-Datei (für sehr klein/simple)
- Redis / Postgres (wenn du später skalieren willst)

---

## Ablauf im Detail

### 1) Setup / Initialisierung (einmalig pro Server)
- Admin-Command z.B.:
    - `/setup weekly_channel:#weekly-trials events_channel:#trials-info`
- Bot postet im `events_channel` **eine** initiale Embed-Nachricht:
    - Speichert `events_message_id` in DB
- Bot bestätigt Setup

---

### 2) Weekly Trials Job (Montag 09:00)
1. `trials = fetchWeeklyTrials()`
2. `embed = buildWeeklyTrialsEmbed(trials)`
3. `sendMessage(weekly_channel_id, embed)`

Optional:
- Erwähne “gültig bis …”
- Ping-Rolle (z.B. `@Trials`) konfigurierbar

---

### 3) Events Overview Refresh Job (laufend)
1. `events = fetchTrialRelevantEvents()`
2. `embed = buildEventsOverviewEmbed(events)`
3. `editMessage(events_channel_id, events_message_id, embed)`

Wichtig:
- Wenn `events_message_id` fehlt (z.B. Nachricht gelöscht):
    - Bot erstellt eine neue Nachricht
    - speichert neue `events_message_id`
- Wenn Permissions fehlen:
    - Log + Admin-Hinweis

---

## Robustheit / Edge Cases
- **Rate Limits:** Caching + nur editieren, wenn Inhalt sich geändert hat (Hash/Vergleich).
- **Quelle down:** Letzten bekannten Stand anzeigen + “Data source unavailable” Hinweis im Footer.
- **Message gelöscht:** Neu erstellen & ID updaten.
- **Channel gelöscht:** Setup ungültig → Admin muss neu setupen.

---

## Permissions (Discord)
Der Bot braucht:
- `Send Messages`
- `Embed Links`
- `Read Message History` (zum Finden/Validieren)
- `Manage Messages` ist *nicht zwingend*, aber hilfreich (für bestimmte Flows)

Für Edit:
- Der Bot kann **seine eigene Nachricht** ohne “Manage Messages” editieren.

---

## Empfohlene Commands
- `/setup weekly_channel events_channel`
- `/trials` (zeigt aktuelle Trials on-demand)
- `/events` (zeigt Events-Übersicht on-demand)
- `/refresh` (Admin: force refresh der Overview)
- `/status` (zeigt Config + letzte Update-Zeit)

---

## Logging & Monitoring
- Logge:
    - letzte erfolgreiche Fetch-Zeit
    - letzte erfolgreiche Post/Edit-Zeit
    - Fehler inkl. HTTP Status / Exception
- Optional:
    - Healthcheck endpoint (wenn gehostet)
    - Discord “admin-log” channel für Fehler

---

## Umsetzungsvorschlag (Milestones)
**Milestone 1:** Bot Grundgerüst + Slash Commands + Config speichern  
**Milestone 2:** Persistente Events-Embed-Nachricht (create + edit)  
**Milestone 3:** Weekly Trials Cron (Montag 09:00 Europe/Berlin)  
**Milestone 4:** Caching, Change-Detection, Fehlerfälle (deleted message, source down)  
**Milestone 5:** Polish: Rollen-Pings, schöne Embeds, i18n (DE/EN)

---

## Beispiel: Scheduling (Konzept)
- Weekly Job:
    - Cron: `0 9 * * 1`
    - TZ: `Europe/Berlin`
- Events Refresh:
    - Every 10 minutes

---

## Definition: “Trials wichtige Events”
Lege fest, welche Events hier rein sollen (Beispiele):
- zeitlich begrenzte Events, die Trials beeinflussen
- Rotations, die Trial-Objectives betreffen
- Start/Ende-Zeiten + Countdown
- “Next rotation” Vorschau
