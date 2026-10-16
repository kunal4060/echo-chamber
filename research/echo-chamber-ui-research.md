# Echo Chamber — UI/UX Deep Research Report
**CTRL+PLAY 2026 / Gamathon · The Gamers Crib · Horror / Audio-Visual Survival**
Prepared: 16 Oct 2026 (UI/UX track). Tech: single HTML file, vanilla JS + Canvas, zero dependencies, fully offline.
Companion: `echo-sequence-research.md` (transferable tricks noted inline as ↳).

**The brief (source of truth):** trapped in a dark abandoned laboratory; a sonar pulse briefly reveals nearby walls/objects/routes, but every pulse makes sound that may attract the hunting creature. Find 3 keycards in different areas, use distractions (thrown objects, radios) to lure the creature away, hide/move quietly, choose when to pulse, reach the exit after all keycards without being captured.

**The direct ancestor:** *Perception* (2017, ex-BioShock devs) — blind protagonist, echolocation visualises the mansion in ghostly blues/whites, the map is entirely black until sound reveals it, and every cane-tap risks attracting the Presence; "sound bombs" work as distractions. Our game is *Perception* compressed into a 2D top-down jam build. Other references: *Darkwood* (vision cone, unseen = vanished), *Alien: Isolation* (motion tracker proximity dread; the tracker itself makes noise), *Amnesia: The Bunker* (Shepard Tone ambient tension), *Yomawari* (heartbeat as proximity warning), *World of Horror* (limitation-as-style, 1-bit dread).

---

## 1. Visual design language — rendering darkness in Canvas

Darkness is not a background colour here; it is the **core mechanic's canvas**. The player must feel blind, not just see a dark theme.

### 1.1 The fog-of-war technique (recommended)
1. Render the whole level (walls, floor, items, entities) to an **offscreen canvas** fully lit, every frame (cheap at jam scale).
2. Maintain a `revealTime` Float32Array, one entry per tile, stamped whenever the sonar ring passes over it (or ambient noise touches it).
3. On the visible canvas: draw the lit scene, then cover it with a black overlay whose alpha per-tile = `f(now - revealTime)` — freshly revealed tiles show at ~full brightness, decaying to black over **~2.5 s**. Implement per-tile alpha via a second offscreen "darkness" canvas: fill black, then punch soft holes with `globalCompositeOperation = 'destination-out'` radial gradients centred on recently-revealed tiles (batch them; 100–200 gradient stamps/frame is fine).
4. **Always-on dim awareness:** punch one permanent small soft hole at the player (radius ~1.2 tiles, leaving ~85% black). Total blindness is unplayable; *Perception* solves this with ambient noise giving faint feedback — our dim hole is the equivalent. It also keeps the player sprite readable at all times.
5. **Vignette:** a static radial-gradient overlay (transparent centre → black corners) for cinematic depth. Draw once per frame, costs nothing.

### 1.2 What the revealed world looks like
- When the sonar ring passes, walls/items render as **pale outlines + faint fill** in cold blue-white (`#BFEFFF` at ~90% → fading), exactly *Perception*'s wintry ghost palette. Outlines read better than fills in brief flashes.
- Unrevealed = pure black. Per *Darkwood*'s rule: **things outside perception simply vanish** — no grey silhouettes, no "fog". The brain fills the gap, and that is where dread lives.
- Keycards get a tiny persistent glint (1px amber sparkle, alpha pulsing slowly) even in darkness once revealed once — a breadcrumb, not a beacon. Radius: only visible within ~2 tiles, so it rewards remembering, not seeing.
- Floor: near-black slate with a faint tile grid visible only during reveal (helps navigation when lit).
- Grain: a cheap film-grain overlay (pre-rendered noise canvas, redrawn at low alpha, offset randomly each frame) sells "abandoned lab" instantly — *Darkwood*'s grainy hand-painted feel, nearly free.

### 1.3 Readability rules (non-negotiable)
- Player, creature eyes, keycards, exit: each gets a **unique silhouette + unique hue**. Never rely on colour alone (colourblind safety).
- Text ≥ 14px, high contrast bone-on-black.
- The exit door is the only thing allowed a warm colour (amber) — it should feel like salvation.

---

## 2. Sonar pulse visualization — the signature moment

This is the game's logo moment; judges will remember it. It must feel **powerful, beautiful, and dangerous**.

### 2.1 The pulse
- On `Space`: an expanding ring spawns at the player — radius 0 → ~7 tiles over **0.8 s**, stroke width shrinking 6px → 1px, alpha 1 → 0. Colour: cold white-cyan core (`#EAF7FF`) with a cyan outer glow (`#7FD4FF`). Smooth expansion (ease-out), **never a strobe flash** (§10).
- As the ring's leading edge crosses each tile, stamp `revealTime`. Walls do **not** block the ring (sound travels; also simpler) — but the creature "hears" it at full strength regardless of walls, which is the price.
- A soft secondary ring 0.15 s behind the first adds depth (two expanding circles, like a real ping echo).
- Faint concentric "after-rings" linger at the max radius for ~0.5 s, then dissolve.

### 2.2 Cooldown & cost (make the danger legible)
- Cooldown **3 s**, shown as a thin circular sweep around a small sonar icon in the HUD (§4). Pressing during cooldown: dull muted click + icon shakes — never silently ignore input.
- Optional tension device (cheap, high value): the HUD icon briefly shows a **red "heard?" flicker** when the pulse reaches the creature's tile — i.e., the game tells you *your ping was heard*. This is the *Alien: Isolation* motion-tracker parallel: the tool that saves you is the tool that kills you, and the game is honest about it.

### 2.3 Ambient micro-reveals
- Footsteps emit tiny reveal stamps (radius 1 tile, alpha 0.25, decay 1 s) — the world faintly breathes around you as you move, *Perception*-style. Quiet movement (§9) halves this.
- This is what makes the game playable without spamming pulse — and every step is still a (small) risk.

---

## 3. Creature representation — scary but readable

**Golden rule from the research: the monster is scarier unseen.** (*Amnesia*, *Darkwood*, and the bridge-horror best-practices all agree: show it briefly, far away, then remove it.)

### 3.1 How it looks (when visible at all)
- The creature is **never fully lit**. When a sonar ring catches it: a dark humanoid silhouette, slightly taller than the player, drawn in near-black against the revealed background — visible as a *hole in the light* — with two ember eyes (`#FF2A1A`, 3px dots). ↳ Transfer from Echo Sequence: translucency + distinct silhouette is the solved readability trick; here inverted (silhouette + eyes instead of ghost tint).
- When very close in darkness (≤2 tiles): the eyes fade in even without sonar — the oldest scare in the book, and it works because the player earned the information by being in danger.
- No detailed sprite needed. Silhouette + eyes + wrong proportions (too tall, too thin) beat any pixel art at this budget — *World of Horror* proves limitation is a style.

### 3.2 Proximity feedback (the *Alien: Isolation* layer)
Since the creature is usually invisible, proximity must be felt through **three channels at once**:
1. **Audio (primary):** heartbeat rate scales with distance — 40 BPM beyond 10 tiles → 120 BPM within 3 tiles — plus growl volume scaling (§7). This is the *Yomawari* heartbeat trick; it works with eyes closed.
2. **Visual (subtle):** screen-edge vignette pulses red, alpha = `clamp(1 - dist/12)`, pulsing on a slow sine (~0.8 Hz — smooth, never blinking, §10). At ≤3 tiles add faint directional chevrons at the screen edge pointing toward the creature (the Isolation tracker's bottom blinker, top-down version).
3. **HUD (minimal):** a thin vertical "dread meter" at the screen's right edge, 4 bands (dark → amber → red), no numbers. Bands, not digits — dread, not data.

### 3.3 Behavioural readability
- When the creature **hears** something: a brief audio cue (head-turn click / low chitter) + it visibly *pauses* if revealed — the player learns "it heard that".
- When it **loses** the player: it slows, sweeps (visible if revealed), then resumes wandering — the player learns "I'm safe for now".
- **Capture sequence (one-shot, never repeated cheaply):** screen cuts to black over 0.15 s → single loud sting → the eyes flash large for ~0.4 s → cut to lose screen (§5). No white flash, no strobe.

---

## 4. HUD — minimal, diegetic where possible

The HUD must never break the darkness. Thin lines, low alpha, corners only.

```
┌──────────────────────────────────────┐
│ ▣ ▣ ▣   FIND KEYCARDS 1/3            │  top-left: keycard slots + objective
│                    ((·))  ◉×2        │  top-right: sonar cooldown + distractions
│                                      │
│              (game)                  │
│                                      │
│                        ▌ dread meter  │  right edge: 4-band proximity
└──────────────────────────────────────┘
```
- **Keycards:** three diamond outlines; filled amber + tiny chime when collected. Objective text updates: `FIND KEYCARDS 2/3` → `REACH THE EXIT →` (with a faint amber arrow at screen edge pointing toward the exit once all 3 are held — the exit is salvation, guide the player to it).
- **Sonar:** small ring icon; sweeping cooldown arc; red "HEARD" flicker when the pulse reaches the creature.
- **Distractions:** radio icons × remaining (start with 3: brief says thrown objects *or* radios — HUD shows one pool, `Q` throws toward aim, `E` drops a radio that loops static).
- **Objective line** fades to 40% alpha after 6 s, reappears on keycard pickup. Never a permanent banner.
- **No minimap.** A minimap would murder the horror. The sonar IS the map.

---

## 5. Screens

### 5.1 Title screen
- Black. Slow expanding sonar rings emanate from centre, revealing fragments of the lab + the title **ECHO CHAMBER** in distressed typewriter type (Courier New bold, wide letter-spacing, slight jitter). Tagline: *"Every ping is a whisper. It is listening."*
- `PRESS ENTER` pulsing slowly (sine, not blink). Controls listed small below. **Photosensitivity note** in small text: "Contains flashing light effects." (required, §10.)
- Background audio: low drone + a distant growl every ~7 s. Sets the tone before a single keypress.

### 5.2 How-to (teach in 30 s, not a manual)
Four lines, max, on the title screen or first overlay:
1. `WASD — move · SHIFT — move quietly`
2. `SPACE — sonar pulse (reveals… but it hears)`
3. `Q — throw distraction · E — drop radio`
4. `Find 3 keycards. Reach the exit. Don't let it touch you.`
Line 2 is the whole game in one parenthesis.

### 5.3 Win screen — "ESCAPED"
- The exit door floods the screen with amber light (the only bright warm moment in the game — earned).
- Stats: time, pulses used, distractions used, times spotted. `R — play again`.
- Keep it brief and warm; contrast with the cold game.

### 5.4 Lose screen — "TAKEN"
- Black screen, the two ember eyes fade out slowly. One-line epitaph contextual to the death: *"It heard your pulse."* / *"It saw you move."* / *"You walked into its path."* — this teaches, not just punishes.
- `R — try again` instantly. No penalty loops (horror best practice: punish but respawn fast).

---

## 6. Palette + typography

**Palette (professional horror, no purple-gradient clichés):**
| Use | Colour |
|---|---|
| Void / unrevealed | `#05070A` |
| Deep slate (floor) | `#0D1117` |
| Bone (text, outlines) | `#E8E4D8` |
| Sonar reveal / ring | `#BFEFFF` → core `#EAF7FF` |
| Keycard / exit amber | `#F5A524` |
| Danger / edge pulse | `#C0392B` |
| Creature eyes | `#FF2A1A` |
| Distraction radio | `#7FB069` (muted signal green) |

Two hues carry meaning: **cold blue-white = information** (sonar, reveals), **amber = objective** (keycards, exit). Red appears ONLY for danger. This discipline is what makes the HUD readable in the dark.

**Typography (offline-safe):** system fonts only — no webfonts (offline requirement).
- Title/HUD headers: `Courier New, monospace`, bold, letter-spacing 4–8px — typewriter/lab-report dread.
- Body/objective: same stack, regular, 14px+.
- Canvas-drawn text with `textBaseline` set; pre-render static labels to offscreen once.

---

## 7. Sound design — 100% WebAudio, synthesized, zero files

Init `AudioContext` lazily on first keypress (autoplay policy) — ↳ same pattern as Echo Sequence. Master gain 0.2, `M` mutes. **Every sound below is an oscillator/noise recipe, no audio files** (offline-safe by construction).

| Event | Recipe |
|---|---|
| **Sonar ping** | sine 880→440 Hz sweep, 0.6 s, exp decay + faint filtered-noise "whoosh" (bandpass ~1200 Hz, 0.4 s) |
| **Ambient drone** | 2 detuned sines (55 Hz + 55.7 Hz) → lowpass 200 Hz, gain LFO 0.05 Hz ±30%. Always on, barely audible. (Amnesia's Shepard-Tone trick, simplified: add a third osc at 110 Hz with slow cyclic gain for unease.) |
| **Footstep (normal)** | 60 ms lowpassed noise burst, gain 0.12 |
| **Footstep (quiet)** | same, gain 0.05, lowpass 400 Hz (duller) |
| **Heartbeat** | 2× sine 55 Hz thumps (lub-dub, 0.28 s apart); rate 40→120 BPM mapped from creature distance; gain scales 0→0.5 |
| **Creature growl** | sawtooth 65 Hz + random LFO (±15 Hz, 3 Hz rate) → lowpass 300 Hz; gain = proximity |
| **Creature chitter (heard something)** | 5× short (40 ms) bandpass-noise bursts at ~2 kHz, accelerating |
| **Creature hunting sting** | dissonant minor-2nd (e.g. 220 + 233 Hz saws), 0.8 s swell — plays once when it locks onto the player |
| **Keycard pickup** | soft 660→990 Hz sine chime, 0.3 s |
| **Distraction throw** | metallic clatter: 4 short square blips, random 800–2000 Hz, 0.25 s |
| **Radio static loop** | looped white noise → bandpass 3 kHz, gain 0.08, while radio active |
| **Exit unlock hum** | warm 132 Hz sine + 198 Hz fifth, gentle, while exit is open |
| **Capture sting** | 110 + 116.5 Hz saws (dissonant cluster) at full gain 1 s + noise crash; then silence |
| **Win** | soft major arpeggio (261→329→392 Hz sines), 1.2 s — relief, not fanfare |

**Mixing rules:** drone always lowest; heartbeat/growl duck the drone slightly when active (sidechain-ish via gain automation — or simply lower drone gain when creature < 6 tiles); stings are LOUD exactly once (anticipation > scare; never two stings back-to-back).

---

## 8. Tutorialization — the first 30 seconds

Scripted opening room (small, creature far away). Contextual one-line prompts, each dismissed by doing the action:

1. `MOVE — WASD` (player walks; footstep micro-reveals teach the dim awareness radius)
2. `PULSE — SPACE` → ring reveals a keycard glint nearby + the prompt: *"(it reveals… but it hears)"*
3. Walk to keycard → `KEYCARD 1/3 — it will guide you out when you have all three`
4. A scripted distant growl + prompt: *`IT HEARD THAT. Move quietly — hold SHIFT.`*
5. Corridor with a radio pickup → `Q — throw to lure it away`

Five beats, zero paragraphs. The player learns the full loop by doing, and the "pulse = danger" lesson lands emotionally in the first minute — which is exactly what the judges need to feel.

---

## 9. Controls

| Input | Action |
|---|---|
| `WASD` / arrows | move (1 tile per press — **recommend turn-based/grid movement** for jam reliability; each move = one creature step) |
| hold `SHIFT` | quiet move (half noise radius, half reveal stamp) |
| `SPACE` | sonar pulse (3 s cooldown) |
| `Q` or **click tile** | throw distraction toward facing / clicked tile (range ~5) |
| `E` | drop radio at feet (loops static 6 s, then dies) |
| `R` | instant restart (no confirm — horror best practice) |
| `M` | mute |
| `ENTER` | confirm / start |

- Mouse aim for throws is the intuitive choice (click = "throw it there"); keyboard `Q` throws in facing direction as fallback. One `keydown` + one `click` listener, `preventDefault()` on game keys (↳ Echo Sequence §4.4 pattern).
- Movement model decision: **grid turn-based** (player acts → noise resolves → creature acts → capture check) is strongly recommended — it makes the stealth tactics legible, the creature AI deterministic, and the whole game debuggable in 3 hours. Real-time would need continuous collision + AI tick tuning the jam can't afford.

---

## 10. Accessibility / safety — photosensitivity

Horror + darkness + pulses = real photosensitivity risk. Handle it properly; judges notice care.

- **WCAG 2.3 rule:** no component flashes more than **3 times in any 1-second period**. Our sonar ring is a *smooth expanding gradient* (0.8 s, ease-out) — compliant by design. Never implement the pulse as on/off strobing.
- **No full-screen white flashes, ever.** Capture = fade to black (0.15 s), not a flash frame. Win = slow amber flood (1.5 s ramp).
- **Screen-edge danger pulse:** slow sine 0.8 Hz, alpha ≤ 0.35 — a throb, not a blink.
- **Title screen warning:** "Contains flashing light effects." one line, always visible pre-game.
- **No colour-only signalling:** keycards = diamond shape + amber; creature = eyes + silhouette; exit = door shape + amber; HUD icons have distinct shapes (§4).
- **Mute (`M`)** from the first screen; captions for critical audio cues (e.g. small text "you hear movement" when the heartbeat first starts — cheap, high value).

---

## 11. Submission requirements & evaluation criteria — the judge's lens

The edited PDF's requirements are not paperwork; each maps to a concrete UI deliverable. Design decisions below are justified against what the judges score.

### 11.1 General Submission Requirements → UI checklist
| Requirement | What the UI must include |
|---|---|
| Launches + stays playable fully offline | **No webfonts, no external assets, no fetch, no modules** — every visual and sound is canvas-drawn or WebAudio-synthesized (§6, §7). Test: double-click `index.html` with Wi-Fi off and airplane mode on. |
| All art/sound/fonts/levels/config bundled | Single `index.html` (inline `<script>`, inline `<style>`). Nothing else needed — the file IS the bundle. |
| No online accounts/cloud/APIs during gameplay | Nothing to log into; no network calls exist. localStorage high-score is optional and offline-safe. |
| Start screen | §5.1 title screen (title, PRESS ENTER, controls, photosensitivity note) |
| Basic instructions | §5.2 four-line how-to + §8 contextual tutorial prompts |
| Playable game loop | Move → pulse → collect 3 keycards → distractions → exit; capture = fail state |
| Win/loss screen | §5.3 ESCAPED + stats; §5.4 TAKEN + contextual epitaph + instant retry |
| Launch instructions | `README.txt` next to the file: "Open index.html in any modern browser (Chrome/Edge/Firefox). No install, no internet. Keyboard required." |
| Tested offline before submitting | Pre-submission ritual: fresh browser profile, Wi-Fi off, devtools Network tab empty, full playthrough win + loss paths |

### 11.2 Evaluation criteria → what the UI contributes to each
- **Creativity (originality + special twist).** The twist here is *seeing = risking*. The UI must make that twist **visible and felt**: the sonar ring is beautiful (reward) and the red "HEARD" flicker + heartbeat ramp is terrifying (price) — both on screen/audio simultaneously. If a judge can't *feel* the tradeoff in 60 seconds, the twist is wasted. §2.2 + §3.2 are the highest-leverage UI work in the whole game.
- **Challenge implementation (assigned mechanics delivered).** Every brief mechanic needs an on-screen proof: 3 keycards → HUD slots (§4); distractions → radio icons + visible lure effect; sonar → ring + reveal + cooldown; "hide/move quietly" → SHIFT with visible noise difference (footstep stamp size); exit-after-keycards → amber exit + edge arrow; capture → TAKEN screen. A judge mentally ticks boxes — give each box a pixel.
- **Gameplay (controls, clarity, balance, engagement, meaningful decisions).** Controls: §9, all discoverable in 30 s (§8). Clarity: the reveal-decay system (§1.1) means the player always knows what they *knew* vs what they *know*. Meaningful decisions: every pulse is a gamble the UI prices honestly (cooldown icon + HEARD flicker). Balance levers the UI exposes: pulse radius/cooldown numbers are HUD-visible, so tuning is visible.
- **Technical execution (stability, performance, correct offline operation).** UI-side: per-tile reveal via batched gradient stamps (no per-pixel loops); pre-rendered grain/vignette; capped particle counts; `requestAnimationFrame` render with discrete logic (↳ Echo Sequence §4 pattern); zero external requests (verify in devtools). The game must never soft-lock: `R` restarts from every state, including mid-capture.
- **Completeness (playable build, clear instructions, satisfying ending).** The arc the UI must deliver: title → 30-s tutorial → full loop → ESCAPED with stats, or TAKEN with a teaching epitaph and instant retry. A "satisfying ending" for horror = the amber flood of the exit (§5.3) — the single warm image in a cold game.

### 11.3 Pre-submission UI checklist (run the night before)
- [ ] Title screen: title, PRESS ENTER, 4-line how-to, photosensitivity note, controls
- [ ] HUD: 3 keycard slots, sonar cooldown, distraction count, objective line, dread meter
- [ ] Sonar ring: smooth expand (no strobe), reveal + 2.5 s fade, cooldown enforced
- [ ] Creature: silhouette + ember eyes on reveal; eyes-only when adjacent in dark; never fully lit
- [ ] Proximity: heartbeat scales 40→120 BPM; edge vignette throbs (≤0.35 alpha, 0.8 Hz); directional chevrons ≤3 tiles
- [ ] Capture: fade-to-black (no white flash) → sting → TAKEN + epitaph + instant R
- [ ] Win: amber flood → ESCAPED + stats (time, pulses, distractions, times spotted)
- [ ] `R` restarts from every state; `M` mutes; arrows/space don't scroll the page
- [ ] Offline test: Wi-Fi off, double-click `index.html`, full win + loss playthrough, devtools Network = zero requests
- [ ] `README.txt` with launch instructions next to the file

---

## Sources & further reference
- Perception (echolocation horror) — Eurogamer: https://www.eurogamer.net/perception-is-a-new-horror-game-from-some-of-the-people-behind-bioshock
- Perception, how it visualises sound — PCGamesN: https://www.pcgamesn.com/perception/perception-release-steam
- Darkwood and the horror of the top-down view — Medium: https://medium.com/@spencer2457/darkwood-and-the-horror-of-the-top-down-view-281a4b9c4c9f
- Darkwood review (vision cone/lighting) — Push Square: https://www.pushsquare.com/reviews/ps4/darkwood
- Alien: Isolation motion tracker design — Rock Paper Shotgun: https://www.rockpapershotgun.com/alien-isolation-pc
- Proximity-monitor design notes (Isolation tracker, R6 Pulse, MGS radar): https://github.com/axamgaming/psx_horror_project/blob/HEAD/docs/PROXIMITY_MONITOR_DESIGN.md
- Amnesia: The Bunker — Shepard Tone tension trick: https://www.gamesradar.com/amnesia-the-bunker-uses-a-sound-illusion-from-scary-movies-and-mario-64-to-quietly-stress-you-out/
- Horror sound design (heartbeat, silence, anticipation): https://www.nme.com/features/sounds-scary-why-horror-game-music-is-the-key-to-unlocking-true-fear-3079713
- World of Horror (1-bit horror aesthetic): https://www.eurogamer.net/world-of-horror-review-atmospheric-retro-dread-as-fleeting-as-a-nightmare
- WCAG 2.3 flash thresholds (photosensitivity): https://www.w3.org/WAI/GL/2010/WD-WCAG20-TECHS-20100128/G19
