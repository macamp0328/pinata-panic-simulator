# Piñata Panic — Mechanics Simulator

A browser-based sandbox for playtesting and tuning the core mechanics of **Piñata Panic** before physical production. This is not a game — it is a configurable simulator that lets a designer rapidly iterate on probabilities and candy distribution.

## How to open it

Open `index.html` in any modern browser (Chrome, Safari, Firefox, Edge). No server, no install, no build step — double-click the file and it works.

## What the simulator does

Each session simulates a full multi-round game of Piñata Panic:

- Players take turns as the **batter** (the one holding the piñata). The batter does not collect candy.
- Each turn, a drop roll determines whether candy falls. If it does, the number of pieces is determined by the current **turn phase**, and each piece is independently assigned to a random non-batter player.
- Starting at a configurable turn, there is an escalating **final break** chance each turn. If it triggers, all remaining candy releases at once and the round ends immediately.
- At the end of each round, points are scored and a detailed summary shows the breakdown for every goal.

### Scoring system

Each round has three public goals and one secret goal per player:

| Goal | Condition | Points |
|---|---|---|
| **Goal A** — Most Candy | Player(s) with the most candy this round | +1 |
| **Goal B** — Most [Color] | Player(s) with the most of the drawn color | +1 |
| **Goal C** — At Least N [Color] | All players holding ≥ N of the drawn color | +1 |
| **Favorite Color** | Player(s) with the most of their secret color | +1 |
| **Licorice Penalty** | Any player holding licorice at round end | −1 |

- Goal B and C colors are drawn fresh each round from a shuffled deck of the active non-licorice colors. When only one non-licorice color is active, Goal B and Goal C will share it regardless of the `goalBCShareColor` setting.
- Favorite Color cards are secretly dealt each round (2 per active non-licorice color, cycling if there are more players than cards). Multiple players may hold the same color.
- The end-of-round summary panel shows enough information to fact-check every point awarded.
- The game-over panel ranks players by total **points** (not candy), with candy as a tiebreaker.

The center piñata graphic is a unicorn that reacts to each event: it shakes during the suspense delay, opens its hatch on a candy drop, and flashes on a final break. Dropped candy pieces appear as colored circles below the piñata after each turn.

The **event log** shows a card for every turn — candy drops display colored piece circles and named player chips; misses are muted; breaks are highlighted in red.

Players get random fun names at the start of each game (Dizzy, Blaze, Mochi, etc.) but can be renamed by clicking their name in the player panel.

## Control panel reference

All tunable variables are in the right-side panel. **None of your changes take effect until you press Restart Game** — a yellow warning banner appears whenever you have unsaved changes. You can tweak as many settings as you like before restarting.

---

### Game Setup

**Players** *(default: 4, range: 2–6)*
How many players are in the game. Each player gets a card on the left showing their candy totals. You can click any player's name to rename them.

**Total rounds** *(default: 3)*
How many rounds make up a full game. Points carry over between rounds; the player with the most points at the end wins (candy is used as a tiebreaker).

**Candy per round** *(default: 50)*
How many pieces of candy the piñata holds at the start of each round. This resets every round regardless of how many pieces were left over.

**Candy colors** *(default: 5, range: 2–5)*
How many candy colors are in the mix. Colors are always drawn from the same ordered list — blue, green, pink, yellow — plus **licorice is always included** as the 5th color regardless of this setting. Setting this to 3 gives blue, green, and licorice. Licorice candy uses a special pickup mechanic (see Scoring section).

**End of round** *(default: release)*
What happens to candy still inside the piñata when all turns run out without a final break:
- **release** — remaining candy drops and distributes to players normally, just without any fanfare
- **lost** — candy disappears and no one gets it (tests what happens when the piñata "survives" a round)

---

### Turn Structure

**Min turns / Max turns** *(default: both 16)*
At the start of each round, the simulator secretly rolls a number of turns between these two values. Setting both to the same number makes every round exactly that long. The rolled count is shown in the round/turn display ("Turn 1 of 16").

> Note: even with 16 turns set, a round can end earlier if the final break triggers (see below).

**Drop chance** *(default: 50%)*
The base probability that any candy falls on a given turn. A 50% chance means roughly half of all turns will produce a drop. Raise this to make candy fall more consistently; lower it to make turns feel more unpredictable.

---

### Turn Phases

Phases let you control *how much* candy falls at different points in a round. Each phase covers a range of turns and sets a min and max number of pieces that can fall on any turn within that window.

| Column | Meaning |
|---|---|
| **Start** | The first turn this phase applies to |
| **End** | The last turn this phase applies to |
| **Min** | Fewest pieces that can fall on a drop turn in this window |
| **Max** | Most pieces that can fall on a drop turn in this window |

The defaults create an escalating feel — early turns drop 1–3 pieces, middle turns drop 2–4, late turns drop 3–5.

You can add phases with the **+ Add phase** button and remove any phase with the **×** button. If two phases cover the same turn, the one lower in the list wins (last match wins, like CSS). A warning appears if any turn in the round isn't covered by any phase.

---

### Final Break

Starting at the **Start turn**, every turn has a chance of triggering a final break — the piñata bursts open and all remaining candy falls at once. The probability starts low and climbs as the round progresses.

**Start turn** *(default: 10)*
The first turn where a final break can happen. Turns before this have zero break chance.

**Start %** *(default: 5%)*
The probability of a break on the very first eligible turn. 5% means it's unlikely but possible early on.

**End %** *(default: 40%)*
The probability of a break on the last possible turn (Max turns). By default, there's a 40% chance the piñata breaks on turn 16 if it hasn't already.

The probability climbs smoothly between these two values across the eligible turns. For example with defaults: 5% on turn 10, ~11% on turn 11, ~17% on turn 12, and so on up to 40% on turn 16.

---

### Scoring

**Licorice pickup chance** *(default: 5%)*
When a licorice piece falls in a drop event, each piece independently rolls against this probability. If the roll passes, the piece is accidentally collected by a random non-batter player and goes into their score (and triggers the −1 licorice penalty at round end). If the roll fails, the piece disappears — it is not held by anyone. The licorice drop counter in the center panel tracks how many licorice pieces have appeared in drops this round.

**Goals B & C share color** *(default: off)*
When enabled, Goal C is forced to use the same color as Goal B — both goals reference the same color. When disabled (default), Goal C draws a different color than Goal B when more than one active non-licorice color is available; if only one exists, both goals share it regardless of this setting.

---

### Drop Timing

**Delay min / Delay max** *(default: 0 sec / 3 sec)*
Controls the suspense window between pressing ACTIVATE and seeing the result. Each turn picks a random delay between these two values (in 0.25-second increments). The piñata shakes during this window. Set both to 0 for instant results during rapid playtesting.

---

### Buttons

**Restart Round** — Resets the current round from turn 1 with a full candy refill. Player scores from completed rounds are kept. Does *not* apply any pending control panel changes.

**Restart Game** — Applies all pending control panel changes, then resets everything: scores, rounds, and candy. Use this after editing the control panel.

**Load Defaults** — Restores every setting to its factory default value and immediately starts a fresh game.

## Candy colors

The simulator uses 5 candy colors: blue, green, pink, yellow, and licorice (black). The **Candy colors** control lets you use 2–5 of them — licorice is always included, and the remaining active colors are drawn from the front of the non-licorice list. Color counts per player are tracked and used for all goal scoring.

Licorice candy (`#2d2d2d`) behaves differently from other colors: it is not freely distributed when it falls. Instead, each piece independently rolls against the **Licorice pickup chance** to determine if a player accidentally picks it up. Uncollected licorice simply disappears.

## Final break mechanics

Starting at the configured start turn, each turn rolls an escalating break chance:

```
breakChance = startPct + (currentTurn − startTurn) / (maxTurns − startTurn) × (endPct − startPct)
```

Default: 5% at turn 10, scaling to 40% at turn 16. When it fires, all remaining candy distributes to non-batter players using the same per-piece random method as a normal drop — the batter is excluded.

## How to share it

Send the `index.html` file. Everything is self-contained — no dependencies, no network calls.
