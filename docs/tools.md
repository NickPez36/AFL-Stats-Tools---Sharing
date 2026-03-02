### Overview of tools

This project currently exposes three analysis tools from the dashboard. Each tool is a standalone HTML page with embedded JavaScript.

- **`AFL_Team_Stats.html`** – Team-level stats and timeline.
- **`AFL_INDV_Stats_v3.0.html`** – Individual Camden player stats and timeline.
- **`AFL_Possession_Tracker_V3.0.html`** – Time-in-possession and field-zone tracker.

Below is a summary of what each tool does, how an analyst typically uses it, and the key data structures involved.

---

### `tools/AFL_Team_Stats.html` – Team-level stats coder

- **Purpose / workflow**
  - Used to code **team-level match events**: entries and exits, scoring chains, stoppages, marks, turnovers, and general play.
  - Typical flow:
    1. Drop a match video on the main drop zone.
    2. Fill in `Match Title`, `Team 1` and `Team 2` names.
    3. Select a quarter (Q1–Q4).
    4. Play the video and click stat buttons (e.g. `Goal - Field Play`, `Inside 50 (Drop Zone)`) at the appropriate time.
    5. Optionally edit or delete events from the canvas timeline.
    6. Export the coded data as a Sportscode XML file, or import an existing XML to review/extend.

- **Key structures and logic**
  - **Stat definitions**
    - `INSTANCE_DURATIONS`: map from stat type string → default clip duration (seconds). Used to infer how much footage around the click should be captured.
    - `STAT_GROUPS`: list of visual stat groups with names, colours, and the stat types they contain. Determines button layout and timeline rows/colours.
  - **Match state**
    - `matchData`: `{ title, team1Name, team2Name, allEvents }`.
    - `allEvents`: array of `{ id, type, quarter, startTimeSeconds, endTimeSeconds }`.
    - Derived structures:
      - `rowMap`: maps group names (e.g. `"Scoring"`, `"Stoppages"`, `"Whistle"`) to row indices on the canvas.
  - **Core functions**
    - `initializeApp()`: wires up drag-and-drop, hotkeys, canvas and scrubber events, and renders buttons.
    - `recordEvent(type)`: validates video and quarter, calculates start/end based on `INSTANCE_DURATIONS`, pushes a new event, and (for drop/rebound inside 50s) also creates an additional `Inside 50` event.
    - `drawTimeline()`: renders grouped rows and all events on the canvas, including a moving playhead synced to `videoPlayer.currentTime`.
    - `loadXML(file)`: parses Sportscode XML `<instance>` nodes into `matchData.allEvents` and infers `team1Name`/`team2Name` from labels with groups `Team 1` / `Team 2`.
    - `generateXML()` / `downloadXML()`: serialises `matchData` to Sportscode XML, including team labels for whistles and `Quarter X` labels for all events.

- **Camden-/season-specific assumptions**
  - Does **not** hard-code Camden Cats by name; intended to be reusable for any two teams.
  - Assumes a fixed **10-minute timeline window** (`TIMELINE_WINDOW_SECONDS = 600`) for the visible canvas segment.
  - Stat types and default durations are hard-coded and should be reviewed for each new season’s coding framework.

---

### `tools/AFL_INDV_Stats_v3.0.html` – Individual stats coder

- **Purpose / workflow**
  - Used to code **individual Camden player actions** against a match video.
  - Typical flow:
    1. Enter a match title (e.g. `Round 1 vs Swans`) in the match setup.
    2. Drop a match video on the video drop zone.
    3. Confirm or edit the default player names in the Camden roster.
    4. Click a quarter (Q1–Q4) to lock the roster, then select the active player card.
    5. While the video plays, click stat buttons (`Kick`, `Handball`, `Mark`, `Inside 50`, `Tackle`, `Hit Out`, `Goal`, `Behind`, `Direct Turnover`) to record events for the selected player and quarter.
    6. Use the whistle button for ball-ups or stoppages not tied to a specific player.
    7. Use the canvas timeline to inspect sequences, edit events, and export or import Sportscode XML.

- **Key structures and logic**
  - **Stat definitions**
    - `STAT_TYPES`: array of stat type strings used to build the stat button grid.
    - Button colours indicate type: Inside 50 (orange), Tackles (red), Direct Turnover (dark), Hit Out (purple), Goals/Behinds (green/yellow).
  - **Match and player state**
    - `matchData`: `{ title, team: { name, players: [...] }, allEvents }`.
    - `team.players`: array of `{ id, name }`, initialised to `Player 1` … `Player 22`.
    - `allEvents`: array of `{ id, type, playerIndex | null, quarter, startTimeSeconds, endTimeSeconds }`.
    - `currentQuarter`: active quarter or `null` if none selected (also controls whether names are editable).
    - `selectedPlayerIndex`: index of the currently selected player card.
  - **Core functions**
    - `initializeApp()`: creates the 22-player roster, renders stat buttons and player cards, and wires drag-and-drop, hotkeys, and timeline behaviour.
    - `renderPlayers()`: renders the player grid, showing selection state and whether names are read-only (when a quarter is active).
    - `setQuarter(quarter)`: toggles the active quarter, controls whether player names are locked, and styles quarter buttons.
    - `recordEvent(type)`: validates that a video, quarter, and (for non-whistle events) player are selected, then creates a new event centred around the current video time (±5 seconds) or a short clip for whistles.
    - `drawTimeline()`: draws rows for `Whistle` plus each player, placing coloured bars labelled with the stat type.
    - `loadXML(file)`: loads roster and events from Sportscode XML:
      - Reads `<row>` codes to infer player names (excluding `Whistle` and stat types).
      - Rebuilds `matchData.allEvents` from `<instance>` nodes, mapping codes back to player indices and using `Stat` and `Quarter` labels.
    - `generateXML()` / `downloadXML()`: exports current events to XML where:
      - Each instance code is either a player name or `Whistle`.
      - A `Stat` label carries the stat type for player events.
      - A `Quarter` label carries `Q1`–`Q4`.

- **Camden-/season-specific assumptions**
  - Hard-coded references to **Camden Cats**:
    - Header text (`Team: Camden Cats` and `Camden Cats Players`).
    - `matchData.team.name` default of `"Camden Cats"`.
    - Default XML download filename prefix `camden-cats-stats`.
  - Assumes a **22-player AFL list** and a **15-minute timeline window** (`TIMELINE_WINDOW_SECONDS = 900`) per view segment.
  - Stat types and roster size are hard-coded but are prime candidates to be read from `config/camden-season.json`.

---

### `tools/AFL_Possession_Tracker_V3.0.html` – Possession and field-position tracker

- **Purpose / workflow**
  - Used to capture **time-in-possession** and **field position** for Camden Cats vs an opposition, using a virtual timer instead of video timestamps.
  - Typical flow:
    1. Optionally set the opposition team name and a video offset (if synchronised later with video).
    2. Click **Start** to begin the virtual match timer.
    3. Select who has possession (`Camden Cats`, `In Dispute`, `Opposition`) and then click zones on the 4×3 field grid to record where the ball is (D50, rebound zones, gold zone, F50).
    4. Use quarter controls (`Start Qtr`, `Whistle`, `End Qtr`) to mark structural events.
    5. Adjust playback speed and use rewind/fast-forward controls when reviewing or correcting data.
    6. Export the session as a Sportscode XML file or import an existing one for review/extension.

- **Key structures and logic**
  - **Timer and playback**
    - `virtualTime`: continuous match time in seconds.
    - `playbackRate`: speed multiplier (1x, 1.25x, 1.5x).
    - `masterUpdate()`: advances `virtualTime` and keeps the playhead in view in the scrollable timeline.
    - `startTimer()`, `togglePause()`, `performReset()`: control timer lifecycle and reset state between sessions.
  - **Possession and zones**
    - `currentQuarter`: `'Q1'`–`'Q4'`, controlled by quarter buttons.
    - `currentPossession`: `"Camden Cats"`, `"In Dispute"`, or the opposition name (from the text input).
    - `zoneButtons`: 12 buttons mapping to zones such as `"D50 Left Pocket"`, `"Gold Zone Middle"`, `"F50 Drop Zone"`.
    - `flipFieldDirection()`: reverses zone labels/data attributes so “attacking” direction changes without altering the underlying logic.
  - **Instances and rows**
    - `instances`: array of `{ start, end, code, labels }` where:
      - `code` is a string like `"Camden Cats - Gold Zone Left"` or `"Start of Quarter"`.
      - `labels` is an array with entries such as `{ group: 'Quarter', text: 'Q1' }` and `{ group: 'Forward Half', text: 'Camden Forward Half' }`.
    - `ROW_DEFINITIONS`: maps logical rows (`Camden Cats`, `In Dispute`, `Opposition`, `Start of Quarter`, `Whistle`, `End of Quarter`) to row indices and colours.
    - `updateTimeline()`: rebuilds the DOM-based timeline, creates rows, positions bars, and keeps the playhead aligned with `virtualTime`.
  - **Core functions**
    - `createInstance(code, labels, isInstant)`: ends the previous open instance, adds a new instance at the current `virtualTime`, and triggers a timeline refresh.
    - Input handlers for possession and zone buttons combine `currentPossession` + selected zone + quarter to:
      - Build `code` and `labels`.
      - Derive `Forward Half` labels based on whether the zone is defensive or forward for Camden vs the opposition.
    - XML import/export:
      - Export writes `<meta name="videoOffset" value="..."/>`, all instances under `<ALL_INSTANCES>`, and row colour definitions under `<ROWS>`.
      - Import reads `<instance>` elements into `instances`, restores `virtualTime` to the latest end time, and reloads timeline and offset from `<meta name="videoOffset">`.

- **Camden-/season-specific assumptions**
  - Hard-coded **Camden Cats** branding:
    - Page `<title>` and header text.
    - Possession button default text and row definitions for Camden vs Opposition.
    - XML filename prefix `camden_cats_analytics_...`.
  - Field zones assume a specific analysis model (D50 vs Gold Zone vs F50) that may evolve between seasons.
  - Quarter structure and rows are fixed, but designed so forward/backwards direction and opposition name can change at runtime.

