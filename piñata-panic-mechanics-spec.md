# Piñata Panic — Mechanics Simulator: Technical Spec

**Version:** 0.4  
**Purpose:** A single-file HTML tool for playtesting and tuning the core game mechanics of Piñata Panic before physical production. Not a full game — a configurable sandbox.

---

## 0. Initial Setup Steps (Before Building)

These steps should be completed before any code is written.

### 0.1 Create the Repository

```bash
git init pinata-panic-simulator
cd pinata-panic-simulator
```

### 0.2 File Structure

The repo only needs a handful of files:

```
pinata-panic-simulator/
├── index.html          ← the single deliverable file
├── README.md           ← project overview for humans
└── CLAUDE.md           ← context file for Claude Code sessions
```

### 0.3 README.md — Contents

The README should cover:
- What this project is (a mechanics simulator, not a game)
- How to open it (just open `index.html` in a browser — no server needed)
- What the control panel does and why it exists
- How to share it (just send the `index.html` file)

### 0.4 CLAUDE.md — Contents

The CLAUDE.md should give Claude Code the context it needs to work on the project in future sessions. Include:

- The purpose of the tool (mechanics tuning sandbox for a tabletop game called Piñata Panic)
- The deliverable constraint: **everything must stay in a single `index.html`** — no splitting into separate CSS/JS files
- The technology constraints: vanilla HTML/CSS/JS ES2022, no frameworks, no CDN, no build tools, evergreen browsers only
- A summary of the core `config` and `state` objects (paste the structures from Section 8 of this spec)
- The phase overlap rule: **last matching phase wins** (see Section 4.3)
- The JS organization rule: logic, render, and event concerns must stay in their respective blocks (see Section 8)
- A note that control panel changes take effect on the next turn, not mid-turn

---

## 1. Overview

This is a browser-based mechanics simulator contained entirely in one `.html` file. The designer opens it locally, adjusts variables in a control panel, presses a button to simulate a turn, and watches what happens. The goal is to rapidly tune probabilities and candy distribution without needing the physical prototype.

---

## 2. Game Structure

- A full game consists of **3 rounds** (configurable)
- Each round, the piñata starts full and is depleted until all candy has fallen or turns run out
- The piñata is **refilled at the start of each new round**
- Players accumulate candy across all rounds — the player with the most candy at the end of the game wins
- Default player count is **4** (configurable, 2–6)

---

## 3. Game State to Track

The simulator must maintain the following state at all times:

- **Current round** — which of the N rounds is active (e.g., Round 2 of 3)
- **Current turn within the round** — turn number (1 through rolled max)
- **Active batter** — which player is currently holding the piñata (rotates each turn)
- **Candy remaining in the piñata** — depletes as candy falls; resets each round
- **Candy collected by each player** — per-round and cumulative totals, plus color breakdown
- **Round status** — active / round-over / game-over

---

## 4. Core Mechanics

### 4.1 Round & Turn Structure

- At the start of each round, roll for how many turns it will last: between `minTurns` and `maxTurns` (defaults: 10–16)
- The rolled turn count is visible to the designer (this is a tuning tool, not a mystery)
- Players rotate as the "batter" each turn in order: Player 1, 2, 3, 4, then back to 1, etc.
- The batter **does not collect candy** — they are the one holding the piñata

### 4.2 Per-Turn Sequence

Each turn runs through the following steps in order:

1. **Final break check** (if current turn >= `finalBreakStartTurn`): Roll against the escalating probability. If triggered, skip to step 5.
2. **Candy drop check**: Roll against `dropChance` (default: 50%). If no drop, log the miss and end the turn.
3. **Candy amount**: Look up the active phase for this turn number (see Section 4.3) and roll for how many pieces fall.
4. **Drop delay**: Randomize a delay between `dropDelayMin` and `dropDelayMax` seconds in 0.25-second increments. The UI enters a suspense state during this window before revealing the result.
5. **Color draw**: Randomly draw the determined number of pieces from the remaining color pool.
6. **Candy distribution**: Distribute all fallen pieces to non-batter players (see Section 4.4).
7. **Advance batter**: Rotate `batterIndex` to the next player.
8. **Check round-over**: If this was the last turn (or final break), proceed to the round summary.

### 4.3 Turn Phases & Candy Amounts

Phases are stored as a **data structure** (array of objects) so the designer can freely add, remove, or edit phases in the control panel without touching any logic:

```javascript
phases: [
  { startTurn: 1,  endTurn: 5,  minCandy: 1, maxCandy: 3 },
  { startTurn: 6,  endTurn: 10, minCandy: 2, maxCandy: 4 },
  { startTurn: 11, endTurn: 16, minCandy: 3, maxCandy: 5 },
]
```

**Phase lookup rule: last matching phase wins.** The code scans the full array and applies whichever phase *last* matches the current turn number. This means if two phases overlap on a given turn, the one defined later in the array takes priority — analogous to CSS cascade. Since phases are naturally written in chronological order (early game first, late game last), a later-defined phase is almost always the intended override. A designer can add a new phase at the bottom of the list to punch through and override any earlier phase for a specific turn window, without special syntax.

If a turn falls outside all defined phases, fall back to the last phase in the array.

The control panel allows the designer to edit each phase's four fields, and add or remove phases. The UI shows a soft warning if there are gaps in turn coverage (e.g., no phase covers turn 9) but does not block saving.

### 4.4 Candy Distribution

When candy falls:
- The active batter is excluded from collection
- Each fallen piece is **independently** assigned to a random non-batter player
- This naturally creates uneven, scramble-like splits without additional logic
- All candy is considered collected by the end of each turn — no uncollected carry-forward state

### 4.5 The Final Break

Starting at `finalBreakStartTurn` (default: 10), each turn includes a separate roll for the piñata fully breaking open. The probability escalates using linear interpolation:

```
breakChance = startPct + (currentTurn - finalBreakStartTurn)
              / (maxTurns - finalBreakStartTurn)
              * (endPct - startPct)
```

Defaults: `startPct` = 5%, `endPct` = 40%, `finalBreakStartTurn` = 10, `maxTurns` = 16.

All three inputs are tunable in the control panel.

When the final break triggers:
- All remaining candy releases at once
- Distributed among non-batter players using the same per-piece random method
- The round ends immediately — go to the round summary screen
- The UI shows a distinct visual state (e.g., "PIÑATA BROKEN" banner, different animation)

### 4.6 Natural End of Round (No Final Break Triggered)

If all turns are exhausted without a final break, the round ends. What happens to candy still inside the piñata is a **configurable option** (`endOfRoundBehavior`):

| Option | Key | Behavior |
|---|---|---|
| Candy auto-releases | `"release"` | All remaining candy drops and distributes to non-batter players, same as a final break but without the dramatic fanfare. |
| Candy is lost | `"lost"` | Remaining candy disappears — no one gets it. Tests what happens when the piñata "survives" a round. |

Default: `"release"`. Exposed in the control panel as a toggle.

### 4.7 Candy Composition

- The piñata holds `candyPerRound` pieces (default: 50), refilled each round
- Split evenly across 5 colors: black, green, yellow, blue, pink
- If the total isn't divisible by 5, distribute the remainder starting from the first color
- Pieces are drawn randomly from the remaining color pool each turn
- Color totals per player are tracked (reserved for future character card scoring)
- All candy colors are worth 1 point each

### 4.8 Round Summary & Between-Round Flow

When a round ends (final break or natural end), the simulator shows a **round summary overlay** that blocks all other interaction. The overlay displays:

- Round number that just completed
- Candy collected this round, per player, ranked highest to lowest
- Cumulative candy totals across all completed rounds, per player, ranked
- If it was the final round: a game winner announcement
- One button: **"Start Round N"** (or **"See Final Results"** after the last round)

The designer must press the button manually — no auto-advance. This gives time to take notes between rounds.

---

## 5. UI Layout

The interface has three main panels plus an overlay:

**Left panel — Players & Scores**
- Lists all players with their names (editable in-place)
- Shows a "BATTER" badge next to the active player
- Displays candy collected this round and cumulative total
- Color breakdown per player (small colored dots with counts)

**Center — Piñata & Turn Controls**
- SVG piñata graphic with visual states (see below)
- Candy remaining indicator: count + simple fill bar
- Round and turn display: "Round 2 of 3 — Turn 5 of 13"
- Visual turn tracker: a row of dots/boxes, color-coded as turns complete (miss = gray, drop = orange, final break = red)
- Large **ACTIVATE** button — the primary interaction, disabled during the delay animation

**Right panel — Control Panel**
- All tunable variables organized by section (see Section 6)
- Reset/restart buttons at the bottom

**Round Summary Overlay**
- Appears on top of everything at round end
- Dismissed only by pressing "Start Round N" or "See Final Results"

### Piñata Visual States

| State | Description |
|---|---|
| Idle | Resting, waiting for input |
| Activated | Button pressed; suspense animation running (e.g., slow shake) |
| Dripping | Partial drop — hatch opens halfway, candy dribbles |
| Broken | Final break — hatch fully open, big candy burst |
| Empty | Round over, piñata visibly depleted |

---

## 6. Control Panel — Tunable Variables

All inputs update live. Changes take effect on the next turn, never mid-turn.

### Game Setup

| Variable | Default | Description |
|---|---|---|
| Number of players | 4 | How many players (2–6) |
| Total rounds | 3 | How many rounds in a full game |
| Total candy per round | 50 | Piñata capacity; resets each round |
| End of round behavior | release | What happens to leftover candy when turns run out: "release" or "lost" |

### Turn Structure

| Variable | Default | Description |
|---|---|---|
| Min turns per round | 10 | Shortest a round can be |
| Max turns per round | 16 | Longest a round can be |
| Base candy drop chance | 50% | % chance candy falls on any given turn |

### Turn Phases (editable list)

Each phase has four editable fields: Start Turn, End Turn, Min Candy, Max Candy. Designer can add or remove phases. Last matching phase wins on overlap (see Section 4.3).

Default phases:
- Phase 1: Turns 1–5, drop 1–3 pieces
- Phase 2: Turns 6–10, drop 2–4 pieces
- Phase 3: Turns 11–16, drop 3–5 pieces

### Final Break

| Variable | Default | Description |
|---|---|---|
| Final break start turn | 10 | Turn at which break rolls begin |
| Final break start % | 5% | Probability at the first eligible turn |
| Final break end % | 40% | Probability at the last possible turn |

Probability interpolates linearly between start % and end % across the eligible turn window (see formula in Section 4.5).

### Drop Timing

| Variable | Default | Description |
|---|---|---|
| Drop delay min (sec) | 0 | Minimum suspense delay |
| Drop delay max (sec) | 3 | Maximum delay (in 0.25-second steps) |

### Buttons

- **Restart Round** — reset turn count and candy for this round; keep cumulative scores
- **Restart Game** — reset everything: scores, round count, candy
- **Load Defaults** — restore all variables to their original values

---

## 7. Event Log

A scrolling log records each turn chronologically:

```
[Round 2 / Turn 3]  Player 2 (batter) — No drop.
[Round 2 / Turn 4]  Player 3 (batter) — 3 pieces fell: 1 green, 2 blue
                    → Player 1 got 2, Player 4 got 1
[Round 2 / Turn 5]  ⚡ FINAL BREAK — 18 pieces released!
                    → Player 2 got 5, Player 3 got 7, Player 4 got 6
```

---

## 8. Implementation Notes for Claude Code

### Tech Stack (Locked)

| Layer | Choice | Notes |
|---|---|---|
| Markup | HTML5 | Standard doctype, semantic elements |
| Styles | Vanilla CSS with custom properties | No preprocessor, no framework, no CDN |
| Scripting | JavaScript ES2022 | Optional chaining (`?.`), nullish coalescing (`??`), `structuredClone()`, `Array.at()` all fair game |
| Graphics | Inline SVG | Piñata and candy graphics defined directly in the HTML — no external image files |
| Browser target | Evergreen (last 2 years) | Chrome, Safari, Firefox, Edge — no IE, no legacy polyfills |
| Build tools | None | File opens by double-clicking — no npm, no bundler, no server |
| External deps | None | No CDN links, no network calls, no localStorage |

### File Structure

Everything in a **single `index.html`**. CSS in `<style>` in `<head>`. JS in `<script>` at the bottom of `<body>`. No separate files.

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Piñata Panic — Mechanics Simulator</title>
    <style>
      /* CSS custom properties (design tokens) at :root */
      /* Layout: CSS Grid for three-panel layout */
      /* Component styles */
    </style>
  </head>
  <body>
    <!-- Left: Player panel -->
    <!-- Center: Piñata + controls + turn tracker -->
    <!-- Right: Control panel -->
    <!-- Overlay: Round summary modal -->
    <script>
      /* 1. Config object */
      /* 2. State object  */
      /* 3. Game logic    */
      /* 4. Render        */
      /* 5. Event listeners */
      /* 6. Init          */
    </script>
  </body>
</html>
```

### JavaScript Organization

The script block is divided into five clearly commented sections. **Do not mix concerns between sections.**

1. **Config** — all tunable defaults. Flat object, no logic.
2. **State** — runtime game state. Never mutated directly by the UI; only by game logic functions.
3. **Game logic** — pure functions that read config/state and return results or mutations. No DOM access.
4. **Render** — `renderAll()` reads state and syncs the entire DOM. All DOM updates happen here and only here.
5. **Events** — `addEventListener` calls that invoke logic functions, then call `renderAll()`.

This separation ensures future Claude Code sessions can find and change behavior without hunting across the file.

### Key Data Structures

```javascript
// Config — all tunable values, driven by the control panel
const config = {
  numPlayers: 4,
  totalRounds: 3,
  candyPerRound: 50,
  endOfRoundBehavior: 'release',   // 'release' | 'lost'
  minTurns: 10,
  maxTurns: 16,
  dropChance: 0.5,
  phases: [
    { startTurn: 1,  endTurn: 5,  minCandy: 1, maxCandy: 3 },
    { startTurn: 6,  endTurn: 10, minCandy: 2, maxCandy: 4 },
    { startTurn: 11, endTurn: 16, minCandy: 3, maxCandy: 5 },
  ],
  finalBreakStartTurn: 10,
  finalBreakStartPct: 0.05,
  finalBreakEndPct: 0.40,
  dropDelayMin: 0,
  dropDelayMax: 3,
  candyColors: ['black', 'green', 'yellow', 'blue', 'pink'],
};

// State — runtime, not directly user-editable
const state = {
  currentRound: 1,
  currentTurn: 1,
  totalTurnsThisRound: null,       // rolled at round start
  batterIndex: 0,                  // index into players array
  candyPool: {},                   // { black: 10, green: 10, ... }
  players: [
    { name: 'Player 1', roundCandy: 0, totalCandy: 0, colorCounts: {} },
    // ...
  ],
  roundOver: false,
  gameOver: false,
  uiPhase: 'idle',                 // 'idle' | 'suspense' | 'result' | 'break' | 'summary'
};
```

### Key Functions

- `initGame()` — build initial state from config, reset everything
- `initRound()` — refill candy pool, roll turn count, reset round-level state
- `takeTurn()` — orchestrate the full per-turn sequence
- `checkFinalBreak(turn)` — compute escalating probability using the linear formula and roll
- `resolveDropChance()` — roll for candy vs. miss
- `getPhaseForTurn(turn)` — scan phases array, return last matching phase (or last phase as fallback)
- `drawCandy(n)` — randomly pull n pieces from the color pool
- `distributeCandy(pieces, batterIndex)` — assign each piece independently to a random non-batter player
- `advanceBatter()` — rotate `batterIndex` mod `numPlayers`
- `renderAll()` — full DOM sync from state
- `logTurn(entry)` — append formatted entry to the event log

### Visual Polish (Nice to Have for v1)

- Piñata shakes during the suspense delay
- Candy pieces animate downward on a drop
- Color-coded dots in event log entries and player panels
- Piñata fill level visually reflects candy remaining
- Distinct flash or banner on final break

---

## 9. Out of Scope (This Sprint)

- Character card bonus scoring (color majority goals)
- Sound effects or audio
- Bluetooth / hardware piñata integration
- Online or networked multiplayer
- Save / export of session data
- Scenario presets beyond "Load Defaults"
