ARC Raiders Bot – Kopierbare Anweisungen für Sonnet

ZIEL
- Nutze die MetaForge ARC Raiders API, um einen Discord-Bot zu bauen, der:
    1) Eine persistente “Trials-Events”-Übersicht als EIN Message-Embed verwaltet (immer editieren).
    2) In dieser Übersicht die relevanten Events (Night Raid, Electromagnetic Storm, Hidden Bunker, Locked Gate, Cold Snap) als einzelne Embeds darstellt: pro Event 1 Embed.
    3) Nur Termine im Zeitfenster 15:00–02:00 (Europe/Berlin) anzeigt.
    4) Stark auf Caching setzt, um API Calls zu minimieren (keine SQL/DB).

API (MetaForge)
- Base URL: https://metaforge.app/api/arc-raiders
- Events Endpoint: GET /api/arc-raiders/events-schedule
- Response: { data: [ { name, map, icon, startTime, endTime }, ... ], cachedAt }

WICHTIG: Die Zeitstempel startTime/endTime sind Unix epoch in Millisekunden (UTC). Für Anzeige/Filter immer nach Europe/Berlin konvertieren.

RELEVANTE EVENTS (Whitelist)
- Night Raid
- Electromagnetic Storm
- Hidden Bunker
- Locked Gate
- Cold Snap

ZEITFENSTER FILTER (15:00–02:00 Europe/Berlin)
- Zeige nur Event-Slots, deren lokaler Start (Europe/Berlin) in folgendem “Nachtfenster” liegt:
    - entweder zwischen 15:00 und 23:59 des Tages
    - oder zwischen 00:00 und 02:00 des Folgetages
- Praktisch: Filterlogik über lokale Uhrzeit (HH:mm) + Datum berücksichtigen.
- Empfehlung: Für “heute” immer ein Fenster bilden:
    - windowStart = heute 15:00 (Berlin)
    - windowEnd   = morgen 02:00 (Berlin)
    - Zeige Events, die im Intervall [windowStart, windowEnd] liegen (StartTime innerhalb oder Overlap; Overlap ist besser).

CACHING (KEINE SQL)
- Nutze 2-stufiges Caching:
    1) In-Memory Cache (schnell)
    2) File Cache (cache.json), damit Neustarts nicht sofort wieder die API spammen

Cache-Key für Events:
- key: "eventsSchedule"
- value: { fetchedAtMs, expiresAtMs, payload, contentHash }
- TTL:
    - Events Schedule TTL: 10 Minuten (oder 5–15 min konfigurierbar)
- Wenn Cache gültig => KEIN API Call.
- Wenn Cache abgelaufen => API Call, Cache neu schreiben.
- Zusätzlicher Schutz:
    - Change detection per Hash (z.B. hash über gefilterte/normalisierte Eventliste ohne “last updated timestamp”)
    - Discord-Edit nur ausführen, wenn sich Hash ändert.

HINWEIS: Die API liefert “cachedAt”. Das ist kein HTTP-ETag, aber du kannst es für Debug/Anzeige nutzen. Trotzdem TTL lokal erzwingen.

AUSGABE / DISCORD EMBEDS
- Es soll “Pro Event ein Embed” geben.
- Alle Embeds werden in EINER Bot-Nachricht gepostet (mehrere Embeds in einer Message) und später immer editiert.
- Reihenfolge der Embeds:
    1) Night Raid
    2) Electromagnetic Storm
    3) Hidden Bunker
    4) Locked Gate
    5) Cold Snap

Embed Inhalt (pro Event)
- Title: "<Event Name>"
- Thumbnail: event.icon (wenn vorhanden)
- Description: Kurz + Zeitfenster z.B. "Termine zwischen 15:00–02:00 (Europe/Berlin)"
- Fields:
    - pro Termin eine Zeile im Field-Value, gruppiert nach Map:
      Format-Beispiel:
      Dam: 18:00–19:00, 21:00–22:00
      Spaceport: 19:00–20:00
      ...
    - oder alternativ: pro Map ein Field (Name=Map, Value=Liste der Times)
- Footer:
    - "Last updated: <Berlin datetime>"
    - optional: "Source cachedAt: <cachedAt>"

Wenn keine Termine im Zeitfenster:
- Embed soll trotzdem existieren:
    - Description: "Keine Termine im Zeitfenster 15:00–02:00."
    - Footer bleibt.

DATENAUFBEREITUNG
1) API payload.data laden
2) Filtern auf name in Whitelist
3) Zeitfenster-Filter anwenden (Berlin)
4) Gruppieren: by (eventName -> map -> [timeRanges])
5) timeRange anzeigen als "HH:mm–HH:mm" in Berlin
6) Sortierung pro Map nach startTime aufsteigend

PERSISTENTE MESSAGE (Events Overview)
- Setup Command: /setup events_channel:#trials-info
    - Bot sendet initial eine Message mit 5 Embeds (auch wenn leer).
    - Speichere events_channel_id und events_message_id in config.json.
- Refresh Job:
    - alle 10 Minuten:
        - getEventsScheduleCached()
        - buildEmbeds()
        - wenn contentHash unverändert => NICHT editieren
        - sonst editMessage(events_channel_id, events_message_id, embeds)

Fehlerfälle:
- Wenn Message gelöscht / nicht gefunden:
    - neue Message posten
    - neue message_id in config.json speichern
- Wenn API down:
    - nutze letzten Cache (auch wenn expired) als Fallback
    - Footer: "Source unavailable – showing last known data"
- Permissions fehlen:
    - loggen und Admin-Hinweis senden (optional)

TRIALS
- Für Trials gibt es noch keine API Doku.
- Implementiere Trials-Teil als Platzhalter/Interface:
    - fetchWeeklyTrials(): TODO
    - buildWeeklyTrialsEmbed(): TODO
- Architektur so bauen, dass Trials später einfach ergänzt werden kann (ohne Refactor).

KONFIG DATEIEN
- config.json (pro guild):
  {
  "guilds": {
  "<guildId>": {
  "events_channel_id": "<id>",
  "events_message_id": "<id>"
  }
  }
  }

- cache.json:
  {
  "eventsSchedule": {
  "fetchedAtMs": 0,
  "expiresAtMs": 0,
  "contentHash": "<hash>",
  "payload": { ...apiResponse }
  }
  }

NON-FUNCTIONAL
- Rate-limit friendly:
    - Keine häufigeren Calls als TTL
    - Keine unnötigen Discord Edits (Hash-Vergleich)
- Timezone muss Europe/Berlin sein (DST beachten).
- Keine SQL/DB.

IMPLEMENTATION HINWEISE (kurz)
- Verwende einen robusten HTTP Client mit Timeout + Retry (z.B. 5s timeout, 1–2 retries).
- JSON parsing muss fehlertolerant sein.
- Logging: letzte erfolgreiche Fetch-Zeit + letzte Edit-Zeit.