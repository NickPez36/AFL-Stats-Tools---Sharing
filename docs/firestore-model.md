### Firestore data model (proposed)

This document sketches a Firestore data model that mirrors the current XML and in-memory structures used by the three tools. The idea is to enable **live data collection** while keeping XML export/import as a first-class feature.

The model assumes a **multi-document, collection-based** design:

- `matches` – one document per match.
- `events` – one document per coded event/instance, linked to a match.
- `teams` – metadata for Camden Cats and opposition clubs (optional but useful).
- `players` – roster information for players (mainly for the individual stats tool).

### Collections and documents

#### `matches` collection

**Path:** `matches/{matchId}`

**Example fields:**

- `season` (number) – e.g. `2026`.
- `round` (number) – e.g. `1`.
- `title` (string) – user-facing title (e.g. `"Round 1 vs Swans"`).
- `teamId` (string) – reference/id for Camden Cats (e.g. `"camden_cats"`).
- `oppositionTeamId` (string) – id for the opposition team (e.g. `"swans"`).
- `createdAt` (timestamp) – when the match doc was created.
- `updatedAt` (timestamp) – last update time.
- `videoOffsetSeconds` (number, optional) – offset between video time and match time (used by the possession tracker).
- `source` (string) – `"xml-import"` | `"live-coding"` | `"mixed"`.

This document can be created when a user starts coding a new match, and its `id` can be passed into all three tools for shared context.

#### `events` collection

**Path:** `matches/{matchId}/events/{eventId}`

Each event corresponds roughly to one `<instance>` in the XML and one entry in the internal arrays (`matchData.allEvents` or `instances`).

**Common fields:**

- `matchId` (string) – redundant but useful for queries (also implied by path).
- `tool` (string) – `"team-stats" | "individual-stats" | "possession-tracker"`.
- `startSeconds` (number) – start time relative to match start.
- `endSeconds` (number) – end time relative to match start.
- `code` (string) – primary code, e.g.:
  - Team stats: `"Goal - Field Play"`, `"Inside 50 (Drop Zone)"`.
  - Individual stats: `"Whistle"` or player name (if mirroring XML) or simply the stat type.
  - Possession: `"Camden Cats - Gold Zone Left"`, `"Start of Quarter"`, `"Whistle"`.
- `quarter` (number) – `1`–`4`, derived from labels or `currentQuarter`.
- `labels` (map<string, string>) – flexible label map mirroring XML labels, e.g.:
  - `"Stat": "Kick"`
  - `"Forward Half": "Camden Forward Half"` or `"Opposition Forward Half"`.
  - Additional future labels as needed.
- `createdAt` (timestamp)
- `updatedAt` (timestamp)

**Tool-specific fields:**

- **Team stats (`tool = "team-stats"`):**
  - `team1Name` (string) – resolved matchData team 1 at time of event creation.
  - `team2Name` (string) – resolved matchData team 2 at time of event creation.
  - Optional `group` (string) – stat group, e.g. `"Scoring"`, `"Stoppages"` (can be derived from `code` using the same mapping as `STAT_GROUPS`).

- **Individual stats (`tool = "individual-stats"`):**
  - `playerIndex` (number) – index into the roster (for efficient queries).
  - `playerName` (string) – resolved at time of event creation (useful if roster later changes).
  - `statType` (string) – e.g. `"Kick"`, `"Goal"`, `"Tackle"` (also stored in `labels.Stat` for XML compatibility).

- **Possession tracker (`tool = "possession-tracker"`):**
  - `teamInPossession` (string) – `"Camden Cats"`, `"In Dispute"`, or opposition name.
  - `zone` (string) – `"D50 Left Pocket"`, `"Gold Zone Middle"`, `"F50 Drop Zone"`, etc.
  - `isQuarterMarker` (boolean) – `true` for `"Start of Quarter"` and `"End of Quarter"` instances.
  - `isWhistle` (boolean) – `true` for whistle markers.
  - `forwardHalfOwner` (string, optional) – `"Camden Cats"` or opposition team id, derived from `Forward Half` label.

### Supporting collections

#### `teams` collection

**Path:** `teams/{teamId}`

Used so Camden Cats and all opposition clubs can share consistent IDs and metadata.

**Example fields:**

- `name` (string) – `"Camden Cats"`.
- `shortName` (string) – `"CAM"`.
- `primaryColor` (string) – e.g. hex colour used for UI.
- `createdAt` / `updatedAt` (timestamps).

#### `players` collection

**Path options:**

- Global: `players/{playerId}` with `teamId` and `season` fields, or
- Scoped: `teams/{teamId}/players/{playerId}` for a team-specific roster.

**Example fields:**

- `teamId` (string) – `"camden_cats"`.
- `season` (number) – e.g. `2026`.
- `number` (number) – guernsey number.
- `displayName` (string) – `"John Smith"`.
- `createdAt` / `updatedAt` (timestamps).

The individual stats tool can keep mapping by `playerIndex` locally, and store a `playerId` or `displayName` into each event document to keep Firestore in sync.

### Mapping from existing tools to Firestore

#### Team stats (`AFL_Team_Stats.html`)

- **Current in-memory shape:** `matchData.allEvents: { id, type, quarter, startTimeSeconds, endTimeSeconds }[]`.
- **XML instances:** `<code>` is `type`, and `Quarter X` is stored in a `<label>`.

**Mapping to Firestore:**

- `startSeconds` ← `startTimeSeconds`.
- `endSeconds` ← `endTimeSeconds`.
- `code` ← `type`.
- `quarter` ← `quarter`.
- `tool` ← `"team-stats"`.
- `labels["Quarter"]` ← `"Quarter " + quarter`.
- Team names can be read from the match doc or from current UI fields at event creation.

#### Individual stats (`AFL_INDV_Stats_v3.0.html`)

- **Current in-memory shape:** `matchData.allEvents: { id, type, playerIndex, quarter, startTimeSeconds, endTimeSeconds }[]`.
- **XML instances:** `<code>` is either `"Whistle"` or the player name; `Stat` and `Quarter` are labels.

**Mapping to Firestore:**

- `startSeconds` / `endSeconds` from `startTimeSeconds` / `endTimeSeconds`.
- `code`:
  - Either reuse `"Whistle"` / player name to stay close to XML, or
  - Use `type` and store player name/index separately.
- `quarter` ← `quarter`.
- `tool` ← `"individual-stats"`.
- `playerIndex` ← `playerIndex`.
- `playerName` ← `matchData.team.players[playerIndex].name` at the time of event.
- `statType` ← `type`.
- `labels["Stat"]` ← `type`.
- `labels["Quarter"]` ← `"Q" + quarter`.

#### Possession tracker (`AFL_Possession_Tracker_V3.0.html`)

- **Current in-memory shape:** `instances: { start, end, code, labels }[]`.
- `code` contains `"Team - Zone"` or `"Start of Quarter"`, `"End of Quarter"`, `"Whistle"`.
- `labels` hold `Quarter` and (optionally) `Forward Half` with text `"Camden Forward Half"` or `"{Opposition} Forward Half"`.

**Mapping to Firestore:**

- `startSeconds` / `endSeconds` from `start` / `end`.
- `code` as-is for maximum XML compatibility.
- `quarter` parsed from `labels.Quarter`.
- `tool` ← `"possession-tracker"`.
- `teamInPossession` parsed from `code` (before `" - "`).
- `zone` parsed from `code` (after `" - "`), when applicable.
- `isQuarterMarker` when `code` includes `"Start of Quarter"` or `"End of Quarter"`.
- `isWhistle` when `code` includes `"Whistle"`.
- `forwardHalfOwner` determined from `labels["Forward Half"]`.

### How tools would use this model

Once Firestore is wired in, the new internal helpers (`eventStore` in the video tools and `instanceStore` in the possession tracker) can:

- **Create event** – write a Firestore document in `matches/{matchId}/events` when `add` / `createInstance` is called.
- **Update event** – update the corresponding document when `update` / `saveEdit` is called.
- **Delete event** – delete the document when `remove` / `clear` or timeline deletions are called.

XML export/import can continue to operate purely on in-memory arrays:

- Export reads from Firestore-backed in-memory arrays and writes Sportscode XML.
- Import reads XML into in-memory arrays and, optionally, **bulk-syncs** to Firestore (e.g. batch write) when the user chooses to push imported data live.

