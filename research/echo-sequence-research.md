# Echo Sequence — Full Research & Design Report
**Gamathon 7 / CTRL+PLAY 2026 · The Gamers Crib · 3-hour offline jam**
Prepared: 16 Oct 2026. Tech plan: single HTML file, vanilla JS + Canvas, zero dependencies.

---

## 1. Game concept explainer

**One line:** A grid puzzle where you record your ghost's walk, then walk beside its replay — both of you must stand on two pressure plates at the same time to unlock the exit.

**Core loop (the 60-second pitch):**
1. **RECORD** — You directly control a glowing ghost ("echo") starting at its own start tile. Walk it (max 12 moves, arrow keys/WASD, Space = wait) to a pressure plate. Press Enter. Your walk is saved as a command list.
2. **PLAYBACK** — Now you control yourself from your start tile. Every command you give (move, wait, even bumping a wall) advances the echo **one step** along its recording. The echo is a translucent replay of what you just did.
3. **SYNC** — Stand on plate B at the same moment your echo stands on plate A. Both plates lit → exit unlocks for your next **4 moves**.
4. **ESCAPE** — Reach the exit before the 4-move window closes. Win screen. Restart (R) or re-record (T) anytime.

**Win:** step onto the exit tile while it is unlocked.
**Lose:** there is no hard fail state — it's a puzzle, not an arcade game. The soft-fail is letting the 4-move window expire (exit re-locks; step off plate B and back on to re-open, since your echo is still parked on plate A). Bad recording? Hit T and record a better one. This trial-and-error loop IS the game — same as its ancestors (see §3).

**Why this works for the jam:** one compact arena, deterministic (zero physics/AI bugs possible), every required mechanic is visible and checkable by judges, and the "aha!" (realising you must WAIT for your echo) lands in under two minutes.

### A note on interpreting the brief (read this first)
The brief says "the player records one movement sequence, then works alongside its replay." Two readings exist:
- **(a) Direct control (RECOMMENDED):** during RECORD you physically walk the echo from its start; the walk becomes the recording. This matches every reference game below, is teachable in 30 seconds, and needs no extra UI.
- **(b) Command tape:** you type a command list that gets replayed from the echo's start, with a ghost preview. Richer puzzle, but needs a tape-input UI + preview renderer — too much for 3 hours, and harder for judges to grasp.

This report designs for **(a)**. Under (a), the rule "blocked moves still consume a command" is implemented as: **during playback, if YOUR move is blocked (you bump a wall), the echo still advances** — which turns wall-bumping into a deliberate timing tool, alongside Space (wait).

---

## 2. Mechanic-by-mechanic breakdown

### 2.1 Recording (cap: 12 commands)
- During RECORD phase, arrow/WASD moves the echo one tile (walls block). Space appends a WAIT command (echo stays, still counts as a command).
- Live HUD: "RECORDING… 7/12" + a 12-slot timeline strip filling with arrow glyphs.
- Backspace = undo last recorded command (cheap to build, huge UX win — make it core, not an extension).
- Enter = finish recording (allowed with 0–12 commands; 0 = echo never moves).
- Trying a 13th command: soft "bonk" sound + timeline flashes. Never silently drop input.
- **Design consequence of the cap:** the echo's route to its plate must be ≤ 12. Our level uses 8 (see §6), leaving comfortable slack.

### 2.2 Playback advancement rule
- Every player command — move, wait, or blocked bump — advances the echo exactly one recorded step, then plate/exit checks run.
- After the recording is exhausted, the echo **parks** (stays on its final tile forever). This is the forgiving heart of the design: once your echo is on its plate, it waits for you patiently.
- Order per step: move player → advance echo → check plates → check exit/win → decrement window.

### 2.3 Simultaneous-plate detection
- After both entities move, check `player on plate B AND echo on plate A` (either entity may be assigned to either plate — symmetric).
- Visual: each plate glows when occupied; when BOTH are occupied, play a chime + particle burst + screen pulse. The player must SEE the sync moment clearly.

### 2.4 The 4-move unlock window
- On sync at step `i`, set `exitOpenUntil = i + 4`. The exit tile is walkable (and shows a countdown "4…3…2…1") for your next four commands.
- Implementation that avoids the classic off-by-one: **never** decrement-then-check in the same step. Check `currentStep <= exitOpenUntil` when the player steps on the exit.
- If the window expires: exit re-locks (red). Recovery is trivial — step off plate B and back on (echo still parked on A) for a fresh window. No punishment, no restart needed.

### 2.5 Restart / re-record
- **R — Restart attempt:** positions reset (player→P, echo→E, replay index→0), recording KEPT, back to playback step 0. Instant, no confirmation.
- **T — Re-record:** recording cleared, back to RECORD phase. This is the "I messed up the plan" button.
- Both must work from every phase. Judges will mash R; it must never soft-lock.

### 2.6 Edge cases (all handled)
1. **Empty recording (0 commands):** echo never moves. Legal; player can still walk around (exit stays locked). No crash.
2. **Player never syncs plates:** free wandering; plates/exit give constant visual feedback. Nothing breaks.
3. **Window expires mid-dash:** exit re-locks; re-sync by stepping off/on B. (Echo parked on A makes this painless.)
4. **Player bumps wall during playback:** player stays, echo still advances. Document in tutorial as a timing trick.
5. **Echo's recorded move blocked at replay:** cannot happen under model (a) — every recorded move was valid when recorded. Keep the guard clause anyway (skip move, consume step).
6. **Recording a wait as the echo:** legal, echo idles N steps during playback. Occasionally useful.
7. **Player on exit tile exactly as window expires:** resolve in player's favour — check win BEFORE window check each step. Feel-good > pedantry.
8. **R pressed mid-record:** discard partial recording, restart record phase cleanly.
9. **What if the player records the echo AWAY from the plate?** Their problem — T exists. The plates are labelled/glowing so intent is clear.
10. **12 vs 13th command:** blocked with feedback, never silent.

---

## 3. Reference-game analysis

### 3.1 The Company of Myself (2009, Eli Piilonen) — the direct ancestor
- **Mechanic:** play a segment, then a time-ghost replays it exactly while you continue; ghosts become teammates (hold switches, act as platforms — one reviewer notes puzzles "require extra-exact timing, learning through trial and error exactly how long to take reaching a certain spot, so that when a clone does it later you're not out of sync").
- **Ghost visuals:** translucent duplicate of the player with a distinct tint; past selves clearly "not you".
- **Transfer lessons:**
  - **Design FOR trial-and-error.** The fun is the retry loop. → Our R key must be instant and unconditional.
  - **Timing is the skill.** → Our wait/bump-to-advance tools are the equivalent of TCOM's "take exactly N seconds" play.
  - Ghost + player physically cooperating (switches) = our plates. TCOM proves the fantasy works.

### 3.2 Cursor\*10 (2008, Nekogames) — minimalism proof
- **Mechanic:** each attempt's cursor is recorded; subsequent attempts play alongside all past cursors; progress by cooperating with yourself (clicking things simultaneously).
- **Visuals:** black-and-white line outlines, isometric — famously crude, famously beloved. "As you layer the progress of your cursors, the beauty of the game reveals itself."
- **Transfer lessons:**
  - **Simple 2D is sufficient** (the brief says so too). Polish = clarity + juice, not assets.
  - Simultaneous-activation puzzles (click two things at once) are fun → our simultaneous plates are the same idea, one step simpler.

### 3.3 The Misadventures of P.B. Winterbottom (2010, The Odd Gentlemen) — recording UX
- **Mechanic:** hold a button to record actions, release → a clone appears looping the recording; an on-screen counter shows allowed clones; whack a clone to remove it; clones are physical (stand on their heads, use as paperweight on switches).
- **Transfer lessons:**
  - **Always show recording state:** Winterbottom's corner counter → our 12-slot timeline strip + pulsing "REC" dot.
  - **Clones must be interactable/visible as objects**, not abstractions → our echo needs a strong silhouette + trail, and plates must react to it visibly.
  - A linear tutorial level that teaches recording before anything else → our 3-step overlay (§5).

### 3.4 Super Time Force (2014, Capybara) — ghost readability at scale
- **Mechanic:** die → rewind → a translucent ghost replays your exact inputs while you try again; ghosts stack; saving a past self has benefits.
- **Transfer lessons:**
  - **Translucency + distinct silhouette** is THE solved way to render "past you". Our echo: ~55% alpha cyan + dotted outline + fading afterimage trail.
  - Even with dozens of ghosts, STF stays readable through colour discipline (ghosts desaturated, player saturated). → Player amber (saturated), echo cyan (ghostly), never similar hues.

### What NOT to take
- **Time Fcuk** (McMillen): dimension-switching, not replay — irrelevant mechanically. (Its monochrome mood is nice, but our palette needs the amber/cyan split for readability.)
- Looping clones (Winterbottom) and death-rewind (STF) are great but out of scope — listed as explicit non-goals for the 3-hour build.

---

## 4. Technical architecture (single-file vanilla JS + Canvas)

No frameworks, no modules (`<script>` inline — `file://` safe), no assets, no network. Turn-based logic; `requestAnimationFrame` used ONLY for cosmetic tweening.

### 4.1 Game states
| State | Meaning | Transitions |
|---|---|---|
| `TITLE` | Title + "press any key" + 3-line how-to | any key → `RECORD` |
| `RECORD` | Player walks the echo; builds command list | Enter → `PLAYBACK` · Backspace = undo · R = restart record |
| `PLAYBACK` | Player moves; echo replays | R = restart attempt (keep recording) · T = re-record · win → `WIN` |
| `WIN` | Overlay: steps used, R/T prompts | R → `PLAYBACK` fresh · T → `RECORD` |

### 4.2 Data structures
- `grid`: 2D array of chars (`#` wall, `.` floor). Entities/plates stored separately, NOT in the grid.
- `player = {x, y, px, py}` — `px/py` are render (tween) positions; `x/y` are logic.
- `echo = {x, y, px, py}` — same.
- `recording = []` — array of `'U'|'D'|'L'|'R'|'W'`.
- `replayIndex`, `phase`, `stepCount`, `exitOpenUntilStep = -1`.
- `plates = {A:{x,y}, B:{x,y}}`, `exit = {x,y}`.
- `P_START`, `E_START` constants.

### 4.3 Core functions (responsibilities, not code)
- `initLevel()` — parse ASCII map, place entities, reset all state.
- `startRecording()` / `recordCommand(cmd)` / `undoCommand()` / `finishRecording()` — record-phase logic; `recordCommand` validates the 12-cap and wall collision for the echo.
- `startPlayback()` — reset positions, `replayIndex = 0`, `stepCount = 0`, phase switch.
- `handleInput(cmd)` — single entry for a player command in PLAYBACK: move player (or bump), then `advanceEcho()`, then `checkPlates()`, then `checkWin()`.
- `advanceEcho()` — if `replayIndex < recording.length`, apply next command (with wall guard), `replayIndex++`.
- `checkPlates()` — if `player∈B && echo∈A` (either plate either entity) and window not already open → `exitOpenUntilStep = stepCount + 4`, effects.
- `checkWin()` — if player on exit and `stepCount <= exitOpenUntilStep` → phase `WIN`.
- `restartAttempt()` — positions/index/window reset, recording kept.
- `render()` → `drawGrid`, `drawPlates` (idle/glow states), `drawExit` (locked red / open green + countdown), `drawEcho` (translucent + trail), `drawPlayer`, `drawHUD` (phase, steps, window), `drawTimeline` (12 slots + playhead), `drawOverlays` (title/win/tutorial).
- `tween()` — each frame, lerp `px/py → x/y` for smooth sliding between tiles (logic stays discrete).
- `sfx(name)` — WebAudio oscillator bleeps (see §5); created lazily on first user gesture (autoplay policy).

### 4.4 Input map
Arrows/WASD = move · Space = wait · Enter = finish recording · Backspace = undo (record) · R = restart · T = re-record · M = mute. `preventDefault()` on arrows/space (page scroll). One `keydown` listener, no key-repeat issues (turn-based = keydown per step is correct; ignore `e.repeat`).

### 4.5 Canvas sizing
Tile 48px; map 11×8 → 528×384 play area. HUD bar on top (~64px), timeline strip below (~56px). Total ~528×504. CSS `max-width: 96vw` + `image-rendering: auto` (smooth tweens, not pixel-art). All text drawn in canvas with a system font stack (no webfonts — offline!).

---

## 5. UI/UX design

### 5.1 Layout
```
┌─────────────────────────────────┐
│ ECHO SEQUENCE   ● REC 7/12      │  ← HUD: title, phase, steps, exit countdown
├─────────────────────────────────┤
│                                 │
│         grid arena              │  ← 528×384 canvas
│                                 │
├─────────────────────────────────┤
│ [↑][↑][→][ ][ ][ ]… (12 slots)  │  ← timeline strip: recording + playhead
├─────────────────────────────────┤
│ Arrows move · Space wait · R…   │  ← controls legend (fades after 20s)
└─────────────────────────────────┘
```

### 5.2 Visual language (the readability system)
- **Player:** warm amber (`#F5A524`) rounded square with a tiny face/direction tick. Saturated, solid — "this is YOU now."
- **Echo:** cyan (`#38E1C6`) at ~55% alpha, dotted white outline, plus 3 fading afterimages trailing its last positions. During RECORD it's slightly more solid (you're driving it); during PLAYBACK it's ghostly (it's a replay).
- **Plates:** brass rings (`#C9A227`) set in the floor; empty = dim; occupied = bright glow + rising particles; both occupied = white-hot pulse.
- **Exit:** a door tile — locked = dark red (`#C0392B`) with a padlock glyph; open = green (`#2ECC71`) pulsing, with big countdown numeral "4·3·2·1" above it.
- **Walls:** bone (`#E8E4D8`) with subtle top-edge highlight — readable, no texture needed. Floor: deep slate (`#14181D`) with faint grid dots.
- **Palette (no purple-gradient clichés):** ink `#101418`, slate `#1B2129`, bone `#E8E4D8`, amber `#F5A524`, cyan `#38E1C6`, brass `#C9A227`, red `#C0392B`, green `#2ECC71`.

### 5.3 Timeline strip (the Winterbottom counter, evolved)
12 slots in a row. During RECORD: slots fill with arrow glyphs (`↑→↓←`) or `·` for wait; current slot pulses. During PLAYBACK: a playhead marker advances per step; used slots dim. The 12-cap is therefore ALWAYS visible — no surprises.

### 5.4 Tutorialization (judge learns in <60s)
- **Title overlay (3 lines, no more):** "① Walk your GHOST to a glowing plate (arrows). ② Press ENTER — now YOU move; every step moves the ghost too. ③ Stand on both plates together, then reach the exit in 4 steps!"
- **Contextual hints (first run only):** stepping on plate B alone → "Good — now WAIT (space) for your echo…"; first sync → "EXIT OPEN! Go go go!"; window expires → "Too slow — step off and back on the plate."
- **Legend bar** under the canvas, fading after ~20s.

### 5.5 Sound (100% offline — WebAudio oscillators, zero files)
| Event | Sound |
|---|---|
| move | short 220Hz square blip (30ms) |
| record command | 330Hz blip |
| wait | soft 180Hz thud |
| bump wall | 120Hz dull knock |
| plate occupied | warm 523Hz ping |
| BOTH plates (sync) | rising arpeggio 523→659→784Hz |
| exit countdown tick | 880Hz tick, rising each step |
| win | little fanfare (523-659-784-1046) |
- Master gain low (0.15), M toggles mute. Init AudioContext on first keydown (browser autoplay rule).

### 5.6 Accessibility / robustness notes
- Shapes differ as well as colours (circle-ish player vs dashed ghost) — colourblind-safe.
- Keyboard-only is fine for the jam; ensure the canvas has focus and arrows don't scroll.
- Text in canvas ≥14px, high contrast.

---

## 6. Level design — one proven arena

### 6.1 The map (11×8)
```
###########
#P..B...X.#
#.........#
###########
###########
###...#####
#E..#..A###
###########
```
- **Top zone (player):** `P`(1,1) → `B`(4,1) → `X`(8,1). Short, open — the player's job is timing, not maze-solving.
- **Bottom zone (echo):** `E`(1,6), winding corridor to `A`(7,6). The corridor is the only floor — no branches, no ambiguity.
- **Divider:** solid wall rows between zones — two clean chambers, one arena.
- Verified by BFS: **E→A = 8 moves** (≤ 12 cap ✓), **P→B = 3**, **B→X = 4** (exactly the unlock window — every step of the dash counts).

### 6.2 Proven solution (simulated step-by-step, script-verified)
**RECORD (8 commands)** — walk the echo: `→ → ↑ → → ↓ → →`
- (1,6)→(2,6)→(3,6)→(3,5)→(4,5)→(5,5)→(5,6)→(6,6)→(7,6) = plate A ✓

**PLAYBACK (12 commands):**
| Step | You do | You at | Echo at | Event |
|---|---|---|---|---|
| 1–3 | → → → | plate B (4,1) | (3,5) | — |
| 4–8 | wait ×5 | plate B | walks → plate A | step 8: **SYNC — exit open (steps 9–12)** |
| 9–12 | → → → → | (5,1)→(6,1)→(7,1)→ **exit X** | parked on A | step 12: **WIN** |

The forced 5 waits ARE the puzzle's "aha": you arrive early and must wait for your past self. Each wait visibly walks the echo closer — satisfying, not boring.

### 6.3 Harder variant ideas (if a level-select is built)
- **"Tight Squeeze":** echo corridor length 11 (one under the cap) — recording must be near-perfect; no wasted moves.
- **"Passing Through":** recording continues PAST plate A, so the echo stands on A for exactly ONE playback step — the player must land on B that exact step. Genuine timing puzzle (no park-and-wait crutch).
- **"Bump Code":** plates placed so the player must use wall-bumps (not Space) to advance the echo — teaches the advanced timing tool.

---

## 7. What's possible — extensions, ranked
| # | Feature | Value | Cost |
|---|---|---|---|
| 1 | Smooth tween sliding between tiles | High (game feel) | Low (lerp in render) |
| 2 | Echo afterimage trail | High (ghost readability) | Low (store last 4 positions) |
| 3 | Backspace undo during record | High (forgiveness) | Tiny |
| 4 | Plate-sync particle burst + screen pulse | High (juice) | Low (canvas particles ~30 lines) |
| 5 | WebAudio bleeps + mute | Medium-high | Low (~40 lines) |
| 6 | localStorage best-steps tracker | Low-medium | Tiny (offline-safe) |
| 7 | 2nd/3rd level select (§6.3 variants) | Medium | Medium (needs verified maps) |
| 8 | Exit countdown numeral + ticking | Medium (tension) | Tiny |
| 9 | Screenshake on sync | Low-medium (juice) | Tiny |
| 10 | Looping-echo mode | High risk — SKIP for jam | — |

**Cut line if behind schedule:** drop 6, 7, 9 first. Never cut 1–4.

---

## 8. 3-hour build plan
| Time | Task | Done when |
|---|---|---|
| 0:00–0:30 | Grid render + player movement + wall collision + input map | Walk around the arena |
| 0:30–1:00 | RECORD phase (echo control, 12-cap, undo, Enter) + PLAYBACK (echo replay, step rule) | Full record→playback loop works |
| 1:00–1:30 | Plates + sync detection + 4-move window + win + R/T | The §6.2 walkthrough wins in-browser |
| 1:30–2:00 | HUD + timeline strip + title/win/tutorial overlays | A stranger can learn it in 60s |
| 2:00–2:25 | Juice: tweens, trail, particles, palette pass | It *feels* good |
| 2:25–2:45 | Sound (bleeps + mute) | Mute works, nothing annoying |
| 2:45–3:00 | Playtest the exact walkthrough, test offline (double-click, no server), write launch instructions | `index.html` opens and wins with zero network |

**If behind:** cut sound to 3 bleeps → cut particles → cut trail. The non-negotiable core is rows 1–3 of this table.

---

## 9. Risks & mitigations
1. **Unsolvable level shipped.** → Mitigated: the §6 map + walkthrough were **verified by script** (BFS distances + full step simulation → WIN at step 12). Never hand-tune the map at the event without re-running the check mentally.
2. **Record/playback desync (echo doesn't match what was recorded).** → Single `advanceEcho()` function, one code path; test by recording the §6.2 walkthrough and watching the echo land on A at step 8.
3. **Judges don't get the mechanic.** → 3-line title overlay + contextual hints (§5.4). If a judge is confused after 60s, the tutorial failed — keep it to three lines.
4. **Exit-window off-by-one.** → Implement as `exitOpenUntilStep = syncStep + 4`, check `step <= exitOpenUntilStep`. (A naive decrement-per-step implementation loses the 4th move — caught during research simulation.)
5. **file:// quirks on the judge's machine.** → No ES modules, no fetch, no webfonts, no external anything. WebAudio initialised on first keypress. Test by literally double-clicking the file with Wi-Fi off.
6. **Arrow-key page scroll / focus loss.** → `preventDefault()` on game keys; click-to-focus not needed (keyboard listener on `window`).

---

## Sources & further reference
- The Company of Myself — Rock Paper Shotgun review: https://www.rockpapershotgun.com/me-myself-i-the-company-of-myself
- Cursor\*10 (playable via Internet Archive): https://archive.org/details/cursor10_flash — RPS on the sequel: https://www.rockpapershotgun.com/cursor10-all-over-again
- P.B. Winterbottom — Eurogamer review (recording mechanic, p.2): https://www.eurogamer.net/the-misadventures-of-p-b-winterbottom-review?page=2
- Super Time Force — RPS design piece (how the time mechanic was built): https://www.rockpapershotgun.com/how-super-time-force-lets-you-play-with-paradoxes
