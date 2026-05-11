# Piñata Panic — Mechanics Simulator

This is a browser-based sandbox for playtesting and tuning the core mechanics of **Piñata Panic** before physical production. It is not a game — it is a configurable simulator that lets a designer rapidly iterate on probabilities and candy distribution.

## How to open it

Just open `index.html` in any modern browser (Chrome, Safari, Firefox, Edge). No server, no install, no build step required — double-click the file and it works.

## What the control panel does

The right-side control panel exposes every tunable variable in the game:

- **Game setup** — number of players, total rounds, candy per round, and what happens to leftover candy when turns run out
- **Turn structure** — min/max turns per round, base candy drop chance
- **Turn phases** — an editable list of turn windows with candy amount ranges (last matching phase wins on overlap)
- **Final break** — the escalating probability that breaks the piñata open, with configurable start turn, start %, and end %
- **Drop timing** — suspense delay range before revealing each turn result

Changes take effect on the **next turn**, never mid-turn.

Buttons at the bottom let you restart the current round (keeping scores), restart the full game, or reload factory defaults.

## How to share it

Just send the `index.html` file. Everything is self-contained — no dependencies, no network calls.
