# Piñata Panic — Mechanics Simulator

A browser-based sandbox for playtesting and tuning the core mechanics of **Piñata Panic** before physical production. This is not a game — it is a configurable simulator that lets a designer rapidly iterate on probabilities and candy distribution.

## How to open it

Open `index.html` in any modern browser (Chrome, Safari, Firefox, Edge). No server, no install, no build step — double-click the file and it works.

## What the simulator does

Each session simulates a full multi-round game of Piñata Panic:

- Players take turns as the **batter** (the one holding the piñata). The batter does not collect candy.
- Each turn, a drop roll determines whether candy falls. If it does, the number of pieces is determined by the current **turn phase**, and each piece is independently assigned to a random non-batter player.
- Starting at a configurable turn, there is an escalating **final break** chance each turn. If it triggers, all remaining candy releases at once and the round ends immediately.
- At the end of each round, a summary shows per-player and cumulative scores.

The center piñata graphic is a unicorn that reacts to each event: it shakes during the suspense delay, opens its hatch on a candy drop, and flashes on a final break. Dropped candy pieces appear as colored circles below the piñata after each turn.

The **event log** shows a card for every turn — candy drops display colored piece circles and named player chips; misses are muted; breaks are highlighted in red.

Players get random fun names at the start of each game (Dizzy, Blaze, Mochi, etc.) but can be renamed by clicking their name in the player panel.

## What the control panel does

All tunable variables are on the right. **Changes take effect when you press Restart Game** — a warning banner appears while you have unsaved changes.

| Section | Controls |
|---|---|
| **Game Setup** | Players (2–6), total rounds, candy per round, candy colors (1–5), end-of-round behavior (release leftover or lose it) |
| **Turn Structure** | Min/max turns per round, base drop chance % |
| **Turn Phases** | Editable list of turn windows with min/max candy amounts. Last matching phase wins on overlap. |
| **Final Break** | Start turn, start %, end % — probability interpolates linearly across the eligible window |
| **Drop Timing** | Suspense delay range (0–10 sec, in 0.25-sec steps) |

**Buttons:**
- **Restart Round** — resets turn count and candy for the current round; keeps cumulative scores
- **Restart Game** — applies any pending control panel changes, then resets everything
- **Load Defaults** — restores all variables to factory defaults and restarts

## Candy colors

The simulator uses 5 candy colors: red, orange, green, blue, and purple. The **Candy colors** control lets you use 1–5 of them (always taken from the front of that list). Color counts per player are tracked for future character-card scoring.

## Final break mechanics

Starting at the configured start turn, each turn rolls an escalating break chance:

```
breakChance = startPct + (currentTurn − startTurn) / (maxTurns − startTurn) × (endPct − startPct)
```

Default: 5% at turn 10, scaling to 40% at turn 16. When it fires, all remaining candy distributes to non-batter players using the same per-piece random method as a normal drop — the batter is excluded.

## How to share it

Send the `index.html` file. Everything is self-contained — no dependencies, no network calls.
