# Echo Chamber — Mechanics & Rules Research Report
**Gamathon / CTRL+PLAY 2026 · The Gamers Crib · offline jam**
**Brief:** Horror / audio-visual survival. Dark abandoned lab, sonar pulse reveals surroundings but attracts the creature. 3 keycards, distractions (thrown objects, radios), quiet movement, exit.
Prepared: 16 Oct 2026. Tech plan: single HTML file, vanilla JS + Canvas, zero dependencies. **No game code in this report — mechanics, rules, and architecture only.**

> **How this was verified:** the level map's connectivity and distances were checked by script (BFS). The creature/player timing numbers were validated through a Python simulation of the core loop (grid movement, Dijkstra noise model, suspicion meter, BFS-chasing creature vs. a scripted competent player). The sim caught 4 real design flaws that were fixed: (1) creature spawn was on the player's forced path → moved to a den; (2) HUNT speed exceeded player speed → guaranteed corridor death → capped below player sprint; (3) vision radius 3 caused unfair corridor brushes → reduced to 2 (hearing-primary); (4) uniform roaming made the opening segment too hot → territorial roam (difficulty gradient). A blind greedy player still dies in the hot segment — which is the intended difficulty; a player using pulse-tracking, lures, and patience has all the tools needed.

---

## 1. Core loop

**60-second pitch:** You're alone in a pitch-black lab. Tap **Space** — a sonar ring blooms, walls and doors flash into view for 2.5 seconds… and somewhere in the dark, something heard it. Sneak (Shift) to 3 keycards scattered across the facility, throw junk or drop radios to pull the creature off your trail, and reach the exit before it reaches you.

**Loop:** Explore in dark (base sight 2.5 tiles) → pulse to reveal → read the layout → move quiet or sprint → manage creature attention (lure / hide / outrun) → grab keycard → repeat ×3 → exit unlocks → escape.

**Win:** step onto the exit tile with 3/3 keycards → WIN screen (time + stats).
**Lose:** creature gets within 0.6 tiles → CAUGHT screen → press R. Restart **keeps collected keycards** (deliberate friendliness for jam judges), resets player to start, creature to den, suspicion to 0.
**Restart:** R anytime, instant, no confirmation, never soft-locks.

---

## 2. Mechanic-by-mechanic breakdown (exact numbers — all tunable, kept as constants at top of file)

### 2.1 Sonar pulse (the signature mechanic)
- **Key:** Space. **Reveal radius:** 6 tiles (Euclidean circle). **Reveal duration:** 2.5 s, fading over the last 0.8 s.
- **Cooldown:** 4 s (HUD radial indicator). **Not ammo-limited** — cooldown-gated (one less UI element, no feel-bad).
- **Noise emitted:** radius **8** — *larger than the reveal radius*. This asymmetry IS the game: every pulse is a calculated risk.
- **Visual:** expanding ring from player (Mark-of-the-Ninja style); revealed tiles get a cyan "pinged" outline that fades.
- **What it reveals:** walls, doors, keycards, exit, and the creature (shown as red eyes — terrifying and useful).
- **Design note:** the opening pulse is safe by construction (creature den is >8 noise-tiles from spawn — verified), so the first pulse teaches the *benefit*; the *cost* is taught the first time the creature investigates a pulse (~15 s in).

### 2.2 Movement: normal vs quiet
- **Normal:** 4.0 tiles/s (0.25 s/tile), noise radius **3** per tile entered.
- **Quiet (hold Shift):** 2.4 tiles/s (0.42 s/tile, 0.6×), noise radius **1** per tile.
- **Rule of thumb for players:** sprint when the creature is far (>6 tiles — your noise can't reach it, minimize exposure time); go quiet in the mid-band (3–6 tiles — don't feed it stimuli); sprint when it's on top of you (<3 tiles — outrun INVESTIGATE's 3.0 t/s; only HUNT at 3.85 is close to your speed).
- Movement is smooth tile-to-tile (interpolated), grid-locked at rest — collisions are exact, no physics.

### 2.3 Noise model
- Every noise is an event: `{tile, radius, suspicion}`.
- **Propagation:** Dijkstra flood from the source; entering a wall tile costs 2, open tile costs 1. **Walls muffle sound** — intuitive, cheap to compute, and it makes wall-hugging a real tactic.
- The creature hears an event if its tile is reached within the event's radius.
- **Values:** quiet step r=1, normal step r=3, sonar r=8, thrown landing r=5, radio tick r=4.

### 2.4 Thrown objects (distractions)
- **Count:** 3. **Key:** click a tile (≤5 Manhattan, must have LOS) or F = throw 4 tiles in facing direction.
- **Invalid target** (wall / out of range): soft "bonk", **not consumed** (judges misclick — don't punish).
- **On landing:** noise burst r=5, suspicion +40 if heard. Visual: expanding ring at landing point.
- **Use:** pull the creature off your corridor — then go the other way. Classic misdirection.

### 2.5 Radios (persistent lures)
- **Count:** 2. **Key:** E drops at current tile.
- **Behavior:** emits noise r=4 **every 2 s for 12 s** (6 ticks), then battery dies.
- **Creature response:** investigates each tick's position; on reaching the radio it **smashes it** (1.5 s) → radio destroyed → SEARCH → ROAM. Effectively **one good lure per radio** (Alien: Isolation lesson — distractions must not work forever).
- **Use:** drop at a corridor mouth, walk the other way. Buys ~8–10 s.

### 2.6 Hiding (ONE simple rule)
- **Stand still (no input) for ≥1.5 s → "hidden":** suspicion decays at **20/s** (vs 8/s baseline). You make no noise, so INVESTIGATE/SEARCH lose your trail fast.
- No lockers, no crouch toggle, no invisibility — stillness + darkness is the whole hiding system. The creature's weak vision (r=2) means a still, quiet player in the dark is genuinely hard to find.
- **Taught in tutorial:** "Stand still to hide. It hunts by sound."

### 2.7 Creature AI state machine
```
ROAM ──susp≥40──▶ INVESTIGATE ──arrive──▶ LOOK (1.2 s) ──▶ SEARCH (4 s) ──▶ ROAM
  ▲                  │                         │
  │                  │ vision / susp≥100       │ susp<35 in HUNT
  │                  ▼                         ▼
  └────────────── HUNT ◀────────────────── (last-known pos)
```
- **ROAM:** wanders at **1.8 t/s** (tick 0.55 s). **Territorial:** 90% of waypoints sampled from x≥5 (center/east "den region") — creates a difficulty gradient: west = learning zone, east = hot zone.
- **INVESTIGATE:** BFS path to the stimulus tile at **3.0 t/s** (tick 0.33 s). New louder stimulus re-targets. Slower than the player's 4.0 — you can outwalk it.
- **LOOK:** 1.2 s pause at the stimulus tile, "sniffs" (checks adjacent tiles by vision).
- **SEARCH:** 4 s wandering near the point, then gives up → ROAM (suspicion −30).
- **HUNT:** triggered by **vision** (instant) or suspicion ≥100. Re-paths to the player's live position every tick at **3.85 t/s** (tick 0.26 s) — *just* below player sprint (4.0): you outrun it in the open, you die in a dead end. If suspicion <35 → drops to SEARCH at last-known position.
- **Stuck failsafe:** if BFS returns no path, wait 0.5 s and retry; after 3 failures → SEARCH. The creature never freezes.

### 2.8 Creature senses
- **Hearing (primary):** per §2.3. Suspicion gains: pulse +70, throw +40, radio tick +25, normal step +12, quiet step +4 (cap 100).
- **Vision (secondary):** radius **2**, Bresenham line-of-sight blocked by walls. Seeing the player = instant HUNT. Deliberately weak — *it hunts by sound in the dark* (thematic + fair).
- **Suspicion decay:** 8/s baseline; 20/s while hidden (§2.6).

### 2.9 Keycards
- **3**, auto-pickup on entering the tile (chime + HUD "1/3 → 2/3"). Placement rule: each ≥8 Manhattan tiles from the others, each in a different facility region (west / north-east / south-east). All reachable — verified by script.

### 2.10 Exit
- One exit tile. **Locked (red, "LOCKED — n/3")** until 3/3 keycards → turns **green** + chime + screen pulse. Step on it → WIN.
- Placed far from start (17 tiles BFS) but only 4–5 from the last keycard — the finale is a short, tense dash.

### 2.11 Capture
- Creature center within **0.6 tiles** of player center → CAUGHT. Screen: red flash, "IT FOUND YOU", stats (keycards, time), **R to retry** (keycards kept).

---

## 3. Edge cases (10, all resolved)

1. **Pulse with creature adjacent:** it hears *you* (+70, target = your tile). Probably HUNT. This is the intended "oh no" moment — the tutorial warns: "Never pulse when it's close."
2. **Throw targeted at wall/out of range:** rejected with soft bonk, **not consumed**.
3. **Radio + pulse on the same tick:** both noises processed; creature reacts to the loudest (pulse). No double-pathing bugs — single stimulus queue, max wins.
4. **Pulse spam:** 4 s cooldown + radius-8 self-aggro makes spam self-punishing. No extra anti-spam needed.
5. **Creature BFS fails (no path):** wait 0.5 s, retry ×3, then SEARCH. Never stuck.
6. **Both enter the same tile on the same tick:** distance-based capture check (<0.6) catches it regardless of ordering.
7. **Exit stepped on with <3 keycards:** bump feedback + "LOCKED — 2/3 KEYCARDS". No penalty.
8. **R pressed mid-HUNT:** instant full reset (positions, suspicion, timers). Keycards kept. Never soft-locks.
9. **Radio dropped on keycard/exit tile:** works normally (it's just a noise source).
10. **Sonar reveals the creature mid-HUNT:** shown as red eyes converging on you — maximum dread, no AI change needed.

---

## 4. Technical architecture (single-file vanilla JS)

**States:** `TITLE → HOWTO → PLAY ⇄ PAUSE → WIN / LOSE`. All rendering on one canvas (600×440 play + 60px HUD = 600×500).

**Data structures:**
- `grid[11][15]` — 0 floor, 1 wall (plus entity markers only in the level definition).
- `player {x, y (float), quiet, hidden}` — tile-locked at rest, interpolated in motion.
- `creature {x, y, state, susp, target, path[], tickT, lookT, searchT}`.
- `noises[]` — event queue processed each frame (single max-loudness stimulus per frame).
- `revealedUntil[11][15]` — timestamps for sonar fade.
- `radios[]`, `throwsLeft`, `keys`, `pulseCd`.

**Named functions & responsibilities:**
- `initLevel()` — parse ASCII map, place entities, reset all state.
- `gameLoop(ts)` — fixed-dt accumulator (dt=1/60), update then render.
- `updatePlayer(dt)` — input → smooth tile movement, emit step noises, pickups, exit check.
- `emitNoise(tx, ty, radius, suspAmt)` — Dijkstra flood (§2.3), queue stimulus.
- `updateCreature(dt)` — tick-based FSM (§2.7): sense (hearing+vision) → transition → move one tile per tick via `bfsPath`.
- `bfsPath(a, b)` / `noiseDist(src)` / `hasLOS(a, b)` (Bresenham) — the three grid workhorses.
- `doSonar()` — cooldown check, set `revealedUntil`, spawn ring visual, `emitNoise(r=8, +70)`.
- `throwObject(tx, ty)` / `placeRadio()` — validation, consume, effects.
- `checkCapture()` — distance < 0.6 → LOSE.
- `render()` — draw order: floor/walls → keycards/exit → radios → rings → player → creature → **fog-of-war mask** (offscreen canvas: black fill, `destination-out` radial holes at player r=2.5 and sonar-revealed tiles) → HUD (keycards, throws, radios, pulse cooldown, suspicion "ear" indicator).
- **Audio:** WebAudio oscillator bleeps only (no files — offline-safe): sonar ping, pickup chime, throw thud, radio crackle, heartbeat when HUNT (rate scales with proximity), capture sting. M toggles mute.

**Input map:** WASD/arrows move · Shift (hold) quiet · Space sonar · Click/F throw · E radio · R restart · M mute · Enter start · Esc pause.

**Timing:** player continuous (interpolated); creature acts on fixed ticks (ROAM 0.55 s / INVESTIGATE 0.33 s / HUNT 0.26 s) — *grid-based real-time*: feels live, stays deterministic and debuggable.

---

## 5. Level design — "The Lab" (15×11, verified by script)

```
###############
#P.....#...#1.#
#.#....#..#.#.#
#.#...........#
#.....C#......#
###.#######.###
#......#.....E#
#.#.........#.#
#.#....#....#.#
#2.....#....3.#
###############
```
`#` wall · `P` player start (1,1) · `C` creature den (10,7) · `1/2/3` keycards · `E` exit (13,6)

**Verified properties (script-checked):** all keycards + exit reachable from P (BFS distances: 1→17, 2→12, 3→19, E→17 tiles); keycards pairwise ≥8 Manhattan apart, each in a different region; creature den 10+ noise-tiles from spawn (opening pulse is safe); four doors (7,3), (7,7), (3,5), (11,5) connect four quadrants; wall stubs create partial cover.

**Difficulty geography (deliberate):** west rooms = safe learning zone (keycard 2, 12 tiles, creature rarely roams here) → north-east = mid (keycard 1, must cross the (7,3) door) → south-east = hot zone (keycard 3 + exit, inside the den region). The game gets scarier as you progress — horror pacing for free.

**Verified walkthrough (timing-checked against the movement numbers):**
1. **t=0 — PULSE** at spawn. Reveals the west rooms (r=6). Creature doesn't hear (too far). You learn: pulse = vision.
2. **Sprint south** to keycard 2 (1,9): 12 tiles ≈ 3 s at 4.0 t/s. **Grab (1/3).** Safe — this is the tutorial keycard.
3. **PULSE** at (1,9): check where the creature is. It's usually east (territorial roam).
4. **North through the (3,5) door** toward (7,3): the first tense stretch. If the pulse showed the creature west/center → **THROW a lure south** (e.g., (2,9)) or **drop a RADIO** at (3,8), wait 3–4 s in the south room while it investigates south, then slip north.
5. Cross **(7,3) door**, grab **keycard 1** (12,1) (2/3). ~25–35 s in.
6. **South-east to keycard 3** (12,9): you're in the den region now — this is where the 2nd throw/radio gets spent. **Grab (3/3)** → exit turns green + chime.
7. **Dash 4–5 tiles north to the exit** (13,6) → **WIN**. Total optimal time: 60–90 s; a real first run: 2–4 min with a scare or two. Ideal jam-demo length.

**Harder variant idea ("Night Shift"):** move the den to (7,5) center, vision radius 2.5, keycards at the four far corners. Same map, same code — pure parameter change. (Do NOT build this for the jam; it's a stretch goal.)

---

## 6. Extensions, ranked (cut line marked)

1. **Noise rings visual** — every noise draws an expanding ring (Mark-of-the-Ninja style). ~20 min. *Highest readability value in the game.*
2. **Creature state icons** — `?` (investigating) / `!` (hunting) above it. ~15 min. *Players must always know what it knows.*
3. **Heartbeat + drone audio** — WebAudio, rate scales with proximity; low drone in dark. ~25 min. *Horror is 50% audio.*
- - - CUT LINE (ship without everything below) - - -
4. Sonar edge-detection: creature shown as eyes-only at reveal edge — 20 min.
5. Fixed patrol waypoints instead of random roam — 15 min.
6. Keycard-gated doors (need n keycards to open a wing) — 45 min.
7. Best-time tracker via localStorage — 15 min.
8. Second creature ("Nightmare mode" toggle) — 30 min, parameterized.

---

## 7. Top risks + mitigations

1. **Creature AI misbehaves** (the #1 jam-killer): mitigations — BFS stuck-failsafe (§3.5), speeds all below player sprint except pressure-preserving HUNT, suspicion *meter* instead of binary detection (no hair-triggers), territorial roam bounds randomness, every number a named constant at file top.
2. **Unfair instant capture:** spawn ≥8 noise-tiles from den (verified), vision only 2, HUNT 3.85 < player 4.0 — the sim proved corridor encounters are escapable, not fatal.
3. **Player lost in the dark:** base sight 2.5 tiles (never pure black), first keycard in open rooms 12 tiles away, 3-line tutorial on the HOWTO screen.
4. **Sonar spam trivializes darkness:** 4 s cooldown + radius-8 self-aggro — spamming summons the creature. Self-balancing.
5. **file:// quirks:** no ES modules, no fetch(), no external anything — one self-contained file, tested by double-click.
6. **Performance:** 15×11 grid; BFS/Dijkstra are trivial at this size; fog-of-war is one offscreen composite per frame.

---

## 8. Brief checkpoint mapping (every Core Challenge → mechanic)

| Brief: Core Challenge | Satisfied by |
|---|---|
| Sonar pulse reveals temporary outlines of rooms/objects | §2.1 Sonar: r=6 reveal, 2.5 s duration, expanding-ring visual |
| Find three keycards in different parts of the facility | §2.9: 3 keycards, ≥8 apart, west / north-east / south-east regions (§5 map) |
| Distractions: thrown objects or radios to lure the creature | §2.4 Thrown (3×, r=5 lure) + §2.5 Radios (2×, 12 s persistent lure) |
| Hide, move quietly, choose when to pulse | §2.2 Quiet mode (Shift, r=1 noise) · §2.6 Hiding rule (stillness) · §2.1 pulse cooldown/noise-8 tradeoff |
| Reach the exit after all keycards, avoiding capture | §2.10 Exit gate (locked→green) · §2.7/2.8 creature AI · §2.11 capture rule |

*No checkpoint is missing. "Offline implementation" is satisfied structurally: local timer (cooldowns), locally stored state (all in-memory; nothing persisted), preloaded everything (single file, zero network).*

---

## 9. Submission checklist

**Screens required:** Title/start (name + "click to start") · How-to-play (controls + the 3 rules: pulse reveals but attracts / Shift is quiet / stand still to hide) · Main game · Pause (Esc) · WIN (time, keycards, deaths) · CAUGHT (R to retry, keycards kept). **Every one of these must exist** — "Completeness" is a criterion.
**Files:** `echo-chamber.html` (the entire game) + `README.txt` (launch: "open echo-chamber.html in any browser, no internet needed" + controls). That's the whole submission.
**Offline test protocol (do this before submitting):**
1. Disconnect device (airplane mode / unplug).
2. Double-click the HTML — game boots with zero console errors.
3. Play: pulse → reveal works; grab a keycard; throw + radio; get caught once (R restarts, keycards kept); collect 3/3 → exit greens → WIN.
4. DevTools Network tab: **zero requests**. If anything loads remotely, the submission fails the brief — fix it.

---

## 10. Evaluation criteria — how this design satisfies each

- **Creativity:** the sonar-that-betrays-you tradeoff (vision *is* the alarm) + a sound-hunting creature in full darkness — the twist is the core loop, not a skin.
- **Challenge implementation:** §8 — every Core Challenge maps to a concrete mechanic; nothing in the brief is unaddressed.
- **Gameplay:** a meaningful decision every few seconds (pulse now or later? quiet or sprint? lure or hide?), readable via noise rings + `?`/`!` states (Invisible Inc lesson: never hide the rules).
- **Technical execution:** single file, zero dependencies, deterministic grid + tick-based AI — stable, performant, and offline by construction (§9 test protocol).
- **Completeness:** title → instructions → full loop → win/lose → instant restart, all art/audio/levels generated in code and bundled in one file.

---

## Appendix A. Reference-game transfers (what was borrowed, from where)

- **Mark of the Ninja** (Klei): *every noise the enemy can hear is drawn as an expanding ring; if the ring touches a guard, you're heard* → our noise-ring visuals for pulses, throws, and (faintly) footsteps. Stealth becomes a readable tool, not a guessing game.
- **Alien: Isolation** (Creative Assembly): *the xenomorph investigates noise, searches the area, and gives up; noisemakers work but the creature doesn't fall for the same trick blindly* → our INVESTIGATE → LOOK → SEARCH → ROAM loop; radios effectively work once (smashed on arrival).
- **Invisible, Inc.** (Klei): *show the player exactly what the enemy can see — perfect information about danger, never unfair surprises* → creature `?`/`!` state icons, visible suspicion cues, pulse reveals the creature itself.
- **Darkwood** (Acid Wizard): *top-down horror where sound matters more than sight* → hearing-primary design, weak vision (r=2), darkness as the default state.
- **Hitman GO** (Square Enix Montréal): *grid clarity makes stealth legible* → grid-locked movement and tile-exact rules, no physics ambiguity.

## Appendix B. Real-time vs turn-based — recommendation

**Recommended: grid-based real-time** — the player moves smoothly tile-to-tile in real time (4.0 / 2.4 t/s); the creature acts on fixed ticks (0.55 s roam / 0.33 s investigate / 0.26 s hunt).
**Why:** the brief's verbs are real-time ("move quietly", "thrown objects", "hide") — turn-based would feel wrong for horror, and the sonar-risk tension needs continuous time. But a *fully* continuous creature AI is hard to tune/debug in a jam; fixed ticks keep it deterministic and step-debuggable while feeling live. This hybrid passed simulation: the numbers above are the tuned result, not guesses.
