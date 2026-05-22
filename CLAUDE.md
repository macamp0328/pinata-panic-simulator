# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This tool is a **mechanics tuning sandbox** for a tabletop game called Piñata Panic. It lets a designer adjust probabilities and candy distribution variables, simulate turns one at a time, and observe outcomes — all without needing the physical prototype.

It is not a polished game. It is a playtesting instrument.

## Deliverable constraint: single file

**Everything must stay in `index.html`.** Do not split into separate CSS or JS files. CSS goes in `<style>` in `<head>`. JS goes in `<script>` at the bottom of `<body>`.

## Technology constraints

- Vanilla HTML/CSS/JS ES2022 — no frameworks, no CDN, no build tools
- Inline SVG for graphics — no external image files
- Light theme using CSS custom properties; no dark-mode media queries
- Evergreen browsers only (last 2 years: Chrome, Safari, Firefox, Edge)
- No network calls, no external dependencies
- `localStorage` is allowed **only** for the preset library (see Presets section); nothing else may use it

## Design tokens (light theme depth stack)

```css
--bg:       #edf1f8;   /* page bg, input fields (recessed)   */
--surface:  #ffffff;   /* panel column backgrounds            */
--surface2: #f4f7fc;   /* cards, event log, overlay           */
--surface3: #e2e8f4;   /* buttons, inactive turn dots         */
--border:   #c4cfe8;   /* dividers, field outlines            */
--text:     #1a2240;
--text-muted: #5870a0;
--accent:   #e94560;   /* red — break events, ACTIVATE btn   */
--accent2:  #d4820a;   /* dark orange — drop events, batter  */
```

## Candy color system

There are always exactly 5 colors in the master palette. `numCandyColors` (2–5) controls how many are active. **Licorice is always the 5th color and is always included** regardless of `numCandyColors`. It cannot be a Favorite Color and is never used in Goal B/C decks.

```js
const COLOR_HEX = {
  blue:     '#1a9fff',
  green:    '#00d26a',
  pink:     '#ff4db8',
  yellow:   '#ffcb2d',
  licorice: '#2d2d2d',
};

function getActiveColors(cfg) {
  const n = cfg.numCandyColors ?? 5;
  const nonLicorice = ['blue', 'green', 'pink', 'yellow'];
  return [...nonLicorice.slice(0, Math.max(0, n - 1)), 'licorice'];
}
```

Always use `getActiveColors(config)` — never read `config.candyColors` directly (it doesn't exist).

## Candy pool composition (per round)

Licorice is always computed first, then remaining candy is split equally among non-licorice colors:

```js
licoriceCount = Math.floor(candyPerRound / 9)   // ~5–6 at 50, ~3 at 25
remaining     = candyPerRound − licoriceCount
perNonLicorice = Math.floor(remaining / activeNonLicoriceColors)  // floor only, no redistribution
```

A few candies may be lost to rounding. This is intentional (fairness guarantee for non-licorice colors).

## Scoring system

Points are tallied at the end of each round via `scoreRound()` and applied via `applyRoundScoring()`.

### Favorite Color cards (secret, per player)
- `dealFavoriteCards()` builds a deck of 2× each active non-licorice color, shuffles, and deals one per player using `deck[i % deck.length]` — cycles if there are more players than cards (possible when few colors are active)
- Stored in `state.favoriteColorCards[]` (index = player index)
- Player(s) with the **most of their own secret color** at round end: +1 pt
- Ties are friendly — all tied players score

### Public Round Goals (3 per round)
- `drawRoundGoals()` returns `{ goalB, goalC, goalCThreshold }`
- **Goal A** (always active): player(s) with most total candy this round → +1
- **Goal B**: shuffle active non-licorice colors, pick one; player(s) with most of that color → +1
- **Goal C**: drawn from separate shuffle (or same as B if `config.goalBCShareColor` is true); falls back to goalB color when only one active non-licorice color exists; all players holding ≥ threshold of that color → +1
- `goalCThreshold = Math.max(2, Math.floor(config.candyPerRound / 10))`

### Licorice penalty
- Any player holding ≥1 licorice at round end: −1 pt (flat, regardless of quantity)
- Licorice is only accidentally collected: each piece rolls `Math.random() < config.licoricePickupChance`

### Score objects
```js
{ goalA, goalB, goalC, favorite, licoricePenalty, total }
// stored in state.roundScoreHistory[].scores[playerIndex]
```

## Core data structures

```javascript
// Config — active values used by the running game
// pendingConfig — what the control panel shows; applied to config on Restart Game
const config = {
  numPlayers: 4,
  totalRounds: 3,
  candyPerRound: 50,
  numCandyColors: 5,
  endOfRoundBehavior: 'release',   // 'release' | 'lost'
  minTurns: 16,
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
  licoricePickupChance: 0.05,  // probability each licorice piece is accidentally collected
  goalBCShareColor: false,     // when true, Goal C color is forced to match Goal B
};

// State — runtime, never mutated directly by UI
const state = {
  currentRound: 1,
  currentTurn: 1,
  totalTurnsThisRound: null,     // rolled at round start
  batterIndex: 0,                // index into players array
  candyPool: {},                 // { blue: 11, green: 11, pink: 11, yellow: 11, licorice: 5 }
  players: [
    { name: 'Dizzy', roundCandy: 0, totalCandy: 0, colorCounts: {}, points: 0, roundPoints: 0 },
    // ... names come from PLAYER_NAME_POOL, not 'Player N'
  ],
  roundOver: false,
  gameOver: false,
  uiPhase: 'idle',              // 'idle' | 'suspense' | 'result' | 'break' | 'summary'
  turnHistory: [],              // [{ type: 'miss'|'drop'|'break' }, ...]
  lastTurnResult: null,         // { type, pieces, dist } — drives last-drop display
  favoriteColorCards: [],       // index = player index, value = color string (never 'licorice')
  roundGoals: { goalB: null, goalC: null, goalCThreshold: 0 },
  droppedLicoriceCount: 0,      // licorice pieces that appeared in drop events this round
  roundScoreHistory: [],        // [{ round, goals, favoriteCards, scores, playerSnapshots }]
};
```

## Pending config pattern

The control panel writes to `pendingConfig`, not `config`. The active game always reads from `config`.

- `hasPendingChanges()` compares `JSON.stringify(config)` vs `JSON.stringify(pendingConfig)`
- A yellow warning banner appears when they differ
- **Restart Game** calls `Object.assign(config, structuredClone(pendingConfig))` then `initGame()`
- **Load Defaults** applies to both objects immediately

## Phase overlap rule

**Last matching phase wins.** The code scans the full phases array and applies whichever phase *last* matches the current turn number — analogous to CSS cascade. A phase added at the bottom overrides any earlier phase for the same turns. If no phase matches, fall back to the last phase in the array.

## Player names

Players get random names from `PLAYER_NAME_POOL` (25 fun names) at game start via `pickRandomNames(count)`. Names are preserved across rounds; `buildPlayers()` only assigns a fresh random name when a player slot has no existing name.

## JS organization rule

The `<script>` block is divided into seven sections. **Do not mix concerns between sections.**

1. **Config** — `COLOR_HEX`, `PLAYER_NAME_POOL`, `DEFAULTS`, `config`, `pendingConfig`, `BELLY_POSITIONS` (precomputed candy dot positions for the belly window)
2. **State** — runtime state object, never mutated directly by UI
3. **Game logic** — pure functions, no DOM access
4. **Render** — `renderAll()` reads state and syncs the entire DOM; all DOM writes happen here. `renderAll()` calls `renderPlayers()`, `renderCenterPanel()`, `renderControlPanel()`. `renderCenterPanel()` calls `renderTurnTracker()`, `renderCandyFill()`, `renderGoals()`, `renderSummaryPanel()`.
5. **Audio** — Web Audio API helpers (`tone()`, `soundSwing()`, `soundMiss()`, `soundCandyPop()`, `soundBreak()`, `soundRoundEnd()`, `soundGameOver()`). Lazy-initializes `AudioContext` on first use.
6. **Events** — `addEventListener` calls that invoke logic, then call `renderAll()`
7. **Init** — `initGame(); initCandyFill(); renderAll();` (`initCandyFill` populates the SVG `#candy-fill` group with pre-positioned circles once; `renderCandyFill` then controls their opacity each render)

## Turn resolution flow

The ACTIVATE button drives a `suspense → result/break → idle` cycle:
1. Button click: `uiPhase = 'suspense'`, button disabled, piñata shakes
2. After a random delay (`dropDelayMin`–`dropDelayMax`): final-break check, then drop check, then candy draw/distribute
3. `uiPhase` set to `'result'` (miss or drop) or `'break'`
4. `finishTurn()` called after non-break turns: advances batter, increments turn, resolves end-of-round if last turn
5. End-of-round triggers a timeout, then `uiPhase = 'summary'`

## Event log

Each turn appends a `.log-card` element (not raw HTML strings). Use `logEntry({ type, round, turn, batterName, pieces, dist, remaining })` where `type` is one of: `'miss'`, `'drop'`, `'break'`, `'end-release'`, `'end-lost'`. The function builds the card DOM directly — do not use `innerHTML` for log entries.

## Control panel timing

Control panel changes go into `pendingConfig` and are **not active** until the user presses **Restart Game**. A warning banner (`#pending-banner`) becomes visible whenever `config` and `pendingConfig` differ.

## Presets

The Presets section at the top of the control panel saves and loads named config snapshots.

- A preset is `{ name, config }` where `config` is a full config object (every key present, including `phases`).
- `SEED_PRESETS` (Config section) holds built-in presets — always present, not editable or deletable.
- User presets persist in `localStorage` under `PRESET_STORAGE_KEY` (`'pinata-panic-presets'`); the module-level `userPresets` array is the in-memory copy. This is the **only** sanctioned `localStorage` use.
- `sanitizeConfig(raw)` fills missing/invalid keys from `DEFAULTS`, so a corrupt or outdated preset can never break the running game. Every preset is passed through it on load/import.
- Loading a preset applies it to `config` and `pendingConfig` and restarts the game immediately. Unsaved control-panel edits are **not** persisted across a refresh — only saved presets are.
- Export downloads the whole user library as `{ version, presets }` JSON; import merges a library file (built-in name collisions are skipped, same-named user presets are overwritten).
