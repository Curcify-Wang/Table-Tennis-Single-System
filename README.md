# Table-Tennis-Single-System
A single Google Apps Script project that unifies the previously separate Knockout (KO) and Round-robin (RR) tournament systems. One spreadsheet can now host multiple KO and RR events at the same time — the menu auto-dispatches each event to the right pipeline based on the Format column on the Event sheet.

## What it is

A single Google Apps Script project that unifies the previously separate **Knockout (KO)** and **Round-robin (RR)** tournament systems. One spreadsheet can now host multiple KO and RR events at the same time — the menu auto-dispatches each event to the right pipeline based on the `Format` column on the **Event** sheet.

---

## Main menu actions (in workflow order)

| # | Menu item | What it does | Backing file |
|---|---|---|---|
| 1 | **Clean Data Book** | Wipes all auto-generated event sheets (keeps `Program` / `Event` / `Players`), so you can start a fresh tournament | [menu.gs](menu.gs) |
| 2 | **Select Events** | Opens a picker dialog to choose which events to process. You can also tick `Use` directly on the Event sheet | [ko_EventSelector.html](ko_EventSelector.html) |
| 3 | **Make List** | Reads players from the Players sheet filtered by sex/class, ranks them by point, and writes one `xxx_list` sheet per event | [ko_makeLists.gs](ko_makeLists.gs) / [rr_makeLists.gs](rr_makeLists.gs) |
| 4 | **Draw by Hand** | KO only: builds an `xxx_hand` sheet, auto-fills seed positions, and opens a sidebar for the referee to drag the rest into place. RR skips this | [ko_makeDraw.gs](ko_makeDraw.gs) |
| 5 | **Draw by Computer** | KO only: runs the ITTF-compliant draw (country balance, half-area control) and writes `xxx_draw` | [ko_DrawAlgorithm.gs](ko_DrawAlgorithm.gs) |
| 6 | **Make Tables / KO, tt** | Reads `xxx_list` / `xxx_draw` and generates: RR's `xxx_group` (round-robin grids) + `xxx_ko` (final KO bracket) + `xxx_tt` (event schedule); for KO, just `xxx_ko` + `xxx_tt` | [rr_makeRR.gs](rr_makeRR.gs) / [ko_makeKO.gs](ko_makeKO.gs) |
| 7 | **Auto Draw** | Runs steps 3–6 end-to-end (KO uses auto seed assignment, no sidebar) | `menu.gs:unifiedAutoDraw` |
| 8 | **Change Pool (RR only)** | When the Players sheet changes after pools have been drawn (sign-ups, withdrawals, class re-classification), this **keeps `groupT` fixed**, preserves existing players' snake order, slots new players into vacated bye positions first, then snake-tails the rest. Rebuilds downstream sheets automatically | `rr_makeLists.gs:rr_changePool_` |
| 9 | **Make TT (auto)** | Merges every event's `xxx_tt` into the master `TT` sheet (time-slot × table-number matrix) | [ko_MatchSchedule.gs](ko_MatchSchedule.gs) |
| 10 | **Make Match (auto)** | Expands `TT` into the `Match` sheet (one row per match: date, time, table, players) | [ko_makeMatch.gs](ko_makeMatch.gs) |
| 11 | **Input Score (auto)** | Opens the score-entry sidebar for referees | [ko_InputScore.gs](ko_InputScore.gs) |
| 12 | **Print Score (auto)** | Generates printable score cards and per-player match schedules | [ko_PrintScore.gs](ko_PrintScore.gs) |

---

## RR submenu (Round-robin only)

| Menu item | What it does |
|---|---|
| **Ranking and Grouping** | Same as Make List, but RR pipeline only |
| **Change Pool (re-snake)** | Same as the main menu's Change Pool |
| **Fill Points & Rank** | After group play, computes each player's points, wins, and rank; writes back into `xxx_group` |
| **Final KO Draw** | After group play, advancers enter a final knockout bracket. Group winners are drawn first; runners-up are placed on the opposite half (half-area separation), with ITTF country balance applied |
| **Simulate Group Scores / KO Results (Test)** | Test mode: fills fake scores to verify downstream pipelines |
| **Medalist** | Builds the medal table |

## KO submenu (Knockout only)

| Menu item | What it does |
|---|---|
| **Check Balance** | After the draw, reports seed and country distribution (debug aid) |
| **Draw by Auto** | Full KO auto-draw chain: list → hand (auto seed) → computer draw → KO tree + tt |

## Setup submenu

| Menu item | What it does |
|---|---|
| **Init Integrated Spreadsheet** | First-time setup: creates the baseline sheets (`Program`, `Event`, `Players`, `TTset`, `TT`, `TX`, `Match`, `score`) |
| **Setup Format Column** | Adds the `Format` column + dropdown (Knockout / Round-robin) to the Event sheet, which is what drives the auto-dispatch |
| **Clean Data Book (KO/RR)** | Per-system wipe (vs. the main menu's Clean Data Book, which wipes both) |

---

## User-edited input sheets

| Sheet | Role |
|---|---|
| **Program** | Global parameters as key/value pairs in columns B/C: BYE rule, seed-to-level, Best Of Games, table-number range, snake-draw mode, etc. |
| **Players** | Master player roster (imported from `data_Common.xls` or edited directly): TournamentID, IPC ID, nation, sex, class, name, point, singles Y/N, etc. |
| **Event** | One row per event: full name, abbreviation, Format (KO/RR), sex, class range, group count, advancers, seed count, `Use` checkbox |

## Auto-generated sheets (one set per event)

Each event uses its abbreviation (e.g. `SM9`, `WS5`) as a prefix:

| Sheet | Contents | Generated by |
|---|---|---|
| `xxx_list` | Entry list and, for RR, the initial snake-grouped pools | Make List |
| `xxx_hand` | KO hand-draw worksheet with seed positions | Draw by Hand |
| `xxx_draw` | Complete KO draw result | Draw by Computer |
| `xxx_group` | RR per-group round-robin grids | Make Tables |
| `xxx_ko` | KO bracket (for RR, this is the final KO for advancers) | Make Tables / Make KO |
| `xxx_tt` | Per-event match schedule | Make Tables |

Aggregate sheets:

| Sheet | Contents |
|---|---|
| **TT** | All events' matches merged (time-slot × table matrix) |
| **TX** | Reverse index of TT (match → time-slot/table) |
| **Match** | Flat list, one match per row, with date/time/table/players |
| **score** | Score-entry cache |
| **Medalist** | Final medal results per event |

---

## KO vs RR — how the pipelines differ

| Stage | KO (single elimination) | RR (round-robin) |
|---|---|---|
| Make List | Plain ranked entry list | Also runs initial snake grouping (with same-country exchange) |
| Draw by Hand | ✅ Hand-draw seeds via sidebar | ❌ Skipped |
| Draw by Computer | ✅ Country-balanced auto-draw | ❌ Skipped |
| Make Tables | Builds the KO bracket + tt | Builds group grids + a follow-up KO + tt |
| Final KO | Not needed (KO is already a bracket) | ✅ Drawn after group play, for advancers |
| Fill Points & Rank | Not needed | ✅ Per-group ranking |

---

## Draw-algorithm highlights

- **Full ITTF compliance**: seed positions, ITTF-handbook BYE placement, country/team area locking, half-area balance
- **Multi-attempt search**: the computer draw runs up to 10 attempts and keeps the one with the fewest country-distribution errors
- **Respects manual edits**: Change Pool deliberately skips the same-country exchange so any manual swaps you made inside `xxx_list` are not overwritten

---

---

## Referee Client

`*Referee Client` spreadsheets are a slim UI for referees. They contain only a thin `Code.gs` that delegates every call to an Apps Script **Library** (e.g. `KOCore`). To change behavior visible to referees, edit the Library project — not the individual client spreadsheets.

## License
MIT License — free for personal, modified, and commercial use. See LICENSE.

## Contact
ilinwang0901@gmail.com
