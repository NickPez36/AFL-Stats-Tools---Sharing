### Overview

This project is a **static, browser-only toolkit** for coding and analysing AFL matches for the Camden Cats. It is served as plain HTML files with embedded JavaScript and Tailwind over a simple static host (no Node/Express backend, no build step).

- **Authentication flow**: `index.html` implements a lightweight login that fetches `config/credentials.json` and, on success, redirects to `dashboard.html`. Credentials are validated entirely in the browser.
- **Dashboard**: `dashboard.html` is a static menu that links directly to each tool in the `tools/` folder via normal `<a>` tags. There is no router or SPA framework.
- **Tools**: Each file in `tools/` is a fully self-contained single-page application (HTML + JS + Tailwind) that runs entirely in the browser, using browser APIs for video, XML, and canvas rendering.

### Shared concepts and data flow

All three tools share a common pattern:

- **Video / time source**
  - A video is either **dropped onto** a drag-and-drop zone or a **virtual timer** is used as the time source.
  - For video-based tools, the current playback time (`HTMLVideoElement.currentTime`) is used to timestamp events.
  - For the possession tracker, an internal virtual clock (`virtualTime` in seconds) advances using `setInterval` and is independent of any video file.

- **Events / instances**
  - Each coded action is represented as an **event instance** with at least:
    - A unique `id` (via `crypto.randomUUID()` in the video tools).
    - A `start` time and `end` time in seconds.
    - A `code` or `type` string (e.g. `"Kick"`, `"Inside 50 (Drop Zone)"`, `"Camden Cats - Gold Zone Left"`).
    - Additional attributes such as `quarter`, `playerIndex`, or label groups.
  - Internally, tools keep events in in-memory arrays:
    - Team stats: `matchData.allEvents`.
    - Individual stats: `matchData.allEvents`.
    - Possession tracker: `instances`.

- **Timeline visualisation**
  - The **team** and **individual** tools use an HTML `<canvas>` to render a horizontal timeline, with:
    - A labelled left margin (rows for stat groups or players).
    - One horizontal row per group or player.
    - Coloured rectangles representing each event’s duration.
    - A vertical red playhead corresponding to current video time.
  - The **possession tracker** uses a scrollable div-based timeline with:
    - Fixed rows for Camden Cats, In Dispute, Opposition, and quarter/whistle markers.
    - Absolutely-positioned coloured bars for each possession instance.
    - A gold playhead that moves with virtual time.

- **Sportscode XML import/export**
  - All tools read and write **Sportscode-compatible XML**:
    - `<ALL_INSTANCES>`: a list of `<instance>` nodes containing `<start>`, `<end>`, `<code>`, and optional `<label>` elements for quarters, stats, teams, or forward-half labels.
    - `<ROWS>`: a list of `<row>` nodes defining each code used in the file (and optionally RGB colour values).
  - Import uses `FileReader` + `DOMParser` to turn XML text into DOM, then the tools recreate their internal event arrays.
  - Export builds XML strings directly from the current in-memory events and triggers a download via `Blob` + `URL.createObjectURL`.

### Entry points

- **`index.html`**
  - Simple login screen.
  - On submit, performs:
    - `fetch('config/credentials.json')` to load stored credentials.
    - Compares input values to `storedCredentials.username` and `.password`.
    - Redirects to `dashboard.html` on success, otherwise shows an inline error.

- **`dashboard.html`**
  - Static landing page listing the three tools:
    - `tools/AFL_Possession_Tracker_V3.0.html`
    - `tools/AFL_INDV_Stats_v3.0.html`
    - `tools/AFL_Team_Stats.html`
  - There is no shared JS across tools; each link opens a separate page and JS runtime.

### Tools at a glance

- **`tools/AFL_Team_Stats.html` – Team-level stats coder**
  - Video-based tool for coding entries, exits, scoring chains, stoppages, marks, turnovers, and general play events by **team** and **quarter**.
  - Uses `INSTANCE_DURATIONS` and `STAT_GROUPS` to define stat types, their default clip lengths, and their visual grouping/colour.
  - Renders events as grouped rows on a canvas timeline and exports/imports team-level Sportscode XML.

- **`tools/AFL_INDV_Stats_v3.0.html` – Individual player stats coder**
  - Video-based tool for coding individual Camden player actions (kicks, handballs, marks, inside 50s, tackles, hit outs, goals, behinds, turnovers).
  - Maintains a roster of players (default 22) and records which player performed each event, by quarter.
  - Shows a per-player timeline on canvas and exports/imports Sportscode XML where rows are players and labels carry stat types and quarter info.

- **`tools/AFL_Possession_Tracker_V3.0.html` – Possession and field-zone tracker**
  - Timer-based tool that tracks which team (Camden, In Dispute, Opposition) has the ball in which field zone (D50, rebound zones, gold zone, F50) over time.
  - Uses a 4×3 field grid UI and a virtual match clock to build possession instances, including quarter markers and whistles.
  - Exports/imports Sportscode XML with time-in-possession instances and colour-coded rows suitable for later visualisation.

### Shared implementation patterns

- **Tech stack**
  - Tailwind CSS loaded via CDN.
  - Google Fonts (Inter or Roboto) via CDN.
  - Vanilla JavaScript modules embedded directly in `<script>` tags.
  - Browser APIs only: no external data services or backend.

- **Config and constants**
  - Each tool currently defines its own constants for:
    - Stat types.
    - Default event durations.
    - Timeline window lengths.
    - Visual rows and colours.
  - These are being progressively refactored to read Camden-specific values (team name, filenames, quarter durations) from `config/camden-season.json` so the tools can be updated between seasons without touching core logic.

