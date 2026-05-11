# Piñata Panic — Mechanics Simulator: Claude Code Context

## Purpose

This tool is a **mechanics tuning sandbox** for a tabletop game called Piñata Panic. It lets a designer adjust probabilities and candy distribution variables, simulate turns one at a time, and observe outcomes — all without needing the physical prototype.

It is not a polished game. It is a playtesting instrument.

## Deliverable constraint: single file

**Everything must stay in `index.html`.** Do not split into separate CSS or JS files. CSS goes in `<style>` in `<head>`. JS goes in `<script>` at the bottom of `<body>`.

## Technology constraints

- Vanilla HTML/CSS/JS ES2022 — no frameworks, no CDN, no build tools
- Inline SVG for graphics — no external image files
- Evergreen browsers only (last 2 years: Chrome, Safari, Firefox, Edge)
- No localStorage, no network calls, no external dependencies

## Core data structures

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

## Phase overlap rule

**Last matching phase wins.** The code scans the full phases array and applies whichever phase *last* matches the current turn number — analogous to CSS cascade. A phase added at the bottom of the list overrides any earlier phase that covers the same turns. If no phase matches, fall back to the last phase in the array.

## JS organization rule

The `<script>` block is divided into five sections. **Do not mix concerns between sections.**

1. **Config** — tunable defaults, flat object, no logic
2. **State** — runtime state, never mutated directly by the UI
3. **Game logic** — pure functions, no DOM access
4. **Render** — `renderAll()` reads state and syncs the entire DOM; all DOM writes happen here
5. **Events** — `addEventListener` calls that invoke logic, then call `renderAll()`

## Control panel timing

Control panel changes take effect on the **next turn**, not mid-turn. The `config` object is read at the start of each turn's logic.
