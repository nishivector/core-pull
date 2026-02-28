# Core Pull — Design Document
Round 17

---

## 1. Identity

**Name:** Core Pull
**Tagline:** *Draw them close. Shatter them against the world.*

**What is the player (specific):**
A magnetized stellar core — dense, ancient, suspended in a copper-oxide void. Not a ship. Not a character. A gravitational fact. Something that *pulls* because that is what it does, and destroys because it cannot help it.

**World feel:**
Two sentences: The world is a tarnished copper crucible — molten at the edges, oxidized and crust-dark at center — where hostile energy fragments orbit endlessly, waiting for something to pull them apart. There is no sky, no ground, no horizon — only the core, the enemies, and the walls they will shatter against.

**Emotional experience:** *Predatory calm breaking into catastrophic release*

**Reference games (DNA notes):**

- **R-Type** — patience before power; the charge mechanic as philosophy; enemies are systems to be read, not reflexes to be beaten
- **Thumper** — every action has weight, every frame is intentional; the world doesn't care about you; rhythm as survival
- **Inside** — environmental dread; beauty in restraint; the world is hostile and gorgeous simultaneously; you are small but singular

---

## 2. Visual Spec

### Colors (exact hex)

| Role | Hex | Use |
|------|-----|-----|
| **Background** | `#7A3E1E` | Full canvas fill — copper/dark terracotta. Never change. Never darken. |
| **Primary** | `#E8A44A` | Player core, UI chrome, score digits |
| **Secondary** | `#C45C1A` | Mid-ground copper grain, idle particle trails |
| **Accent** | `#FFD580` | Blast wave ring, attraction beam tendrils |
| **Enemy body** | `#3B1A0A` | Dark ember, nearly silhouette against copper BG |
| **Enemy glow** | `#FF6B35` | Enemy core heat, hostile pulse |
| **Wall surface** | `#5C2A0E` | Arena boundary — deeper copper, clearly distinct from BG |
| **Wall impact flash** | `#FFFFFF` | 3-frame white flash on enemy wall collision |
| **Attraction beam** | `#FFD58088` | 50% alpha gold filament, animated |
| **Blast ring** | `#FFD580` → `#FF6B3500` | Gradient ring expanding outward, fades to transparent |

### Bloom
- **Strength:** 1.4
- **Threshold:** 0.55
- Applied to: player core, enemy glows, blast wave ring, attraction beam tendrils
- Implementation: additive canvas layer or Pixi.js filter — bloom must be visible on copper background (not just on dark BG)

### Camera
- **Type:** Fixed orthographic, top-down
- **Position:** Centered on canvas, no scroll, no follow
- **Viewport:** 800×800px logical (scales to device)
- **Shake:** On blast release — 6px offset, 180ms decay, sine easing
- **Player is always canvas center** — the world does not move; the player is fixed

### Player silhouette (5 words):
**Dense ringed pulsing magnetic orb**

Specifics: Two concentric rings (outer: 28px radius, inner: 18px radius) rotating in opposite directions at 0.4 rad/s and -0.6 rad/s respectively. Core circle: 12px radius, filled `#E8A44A`. Rings: 2px stroke `#E8A44A`. Ring segments: 8 arc gaps of 12° each, creating dashed-ring appearance.

---

## 3. Sound Spec

### Music: 90 BPM

**Description:** Industrial copper-pulse rhythm. A single bass oscillator (`Sawtooth`, tuned to A1, 55Hz) cycles every 2 beats — slow, tectonic. Over it: a high synth drone (`Sine`, C5, 523Hz, very quiet, 8% volume) creates unease. At beat 1 of every 4-beat bar: a dry metallic `MembraneSynth` kick at 90 BPM. Sixteenth-note hi-hat ghost pattern (`MetalSynth`, very short decay 0.04s, 40% volume) enters at level 2. At level 3+: a pulsing `PWM` pad enters on beats 2 and 4. No melody — only rhythm, weight, and tension. Music is generated procedurally via Tone.js.

### Music State Changes (6 triggers)

| # | Trigger | Change |
|---|---------|--------|
| 1 | **Game start** | Bass oscillator fades in over 2 bars (2.66s); kick pattern begins |
| 2 | **Hold begins (pointerdown)** | Hi-hat pattern doubles to 32nd notes; bass pitch bends down 2 semitones over 0.3s |
| 3 | **Hold >800ms (charge building)** | Drone sine rises from C5 to E5 over 0.5s; subtle reverb wet increases from 0.1 to 0.4 |
| 4 | **Blast release (pointerup)** | Single `MembraneSynth` impact hit at 150Hz, 0.2s decay; bass pitch snaps back; 1-bar percussion fill |
| 5 | **Enemy wall-kill** | `MetalSynth` sharp ring (800Hz, 0.08s decay) pitched up +1 semitone per consecutive kill (resets after 3s) |
| 6 | **Player death** | All music stops; single descending `Sine` glide from 220Hz to 55Hz over 1.2s; silence |

### 6 SFX (exact Tone.js hints)

| # | Event | Tone.js Implementation |
|---|-------|----------------------|
| 1 | **Attract begin** | `Tone.Synth({ oscillator: { type: 'sine' }, envelope: { attack: 0.3, decay: 0, sustain: 1, release: 0.2 } })` — held note at 110Hz (A2), volume ramps from -40dB to -18dB over 300ms |
| 2 | **Attract building** | `Tone.LFO({ frequency: 4, min: -22, max: -16 })` modulates the attract synth volume — creates tremolo wobble that speeds up with hold duration (LFO freq → 8Hz at 800ms hold) |
| 3 | **Blast release** | `Tone.MembraneSynth({ pitchDecay: 0.08, octaves: 4 }).triggerAttackRelease('C2', '8n')` + `Tone.Distortion(0.6)` — deep percussive thud with crunch |
| 4 | **Enemy wall impact** | `Tone.MetalSynth({ frequency: 600, envelope: { attack: 0.001, decay: 0.08, release: 0.1 }, harmonicity: 5.1, modulationIndex: 32 })` — short metallic clang; pitch varies ±100Hz randomly per hit |
| 5 | **Enemy attracted (entering pull radius)** | `Tone.Synth({ oscillator: { type: 'triangle' } }).triggerAttackRelease('E4', '32n')` — soft, quick ping at entry; volume -28dB |
| 6 | **Player death** | `Tone.Synth({ oscillator: { type: 'sawtooth' }, envelope: { attack: 0.01, decay: 1.2, sustain: 0, release: 0 } }).triggerAttack('A2')` then `frequency.rampTo(27.5, 1.2)` — sawed glide to subsonic, with `Tone.Reverb(4)` wet=1 |

---

## 4. Mechanic Spec

### Core Loop (one sentence):
Hold to magnetically pull enemies toward your core, release at the exact moment enemies are close enough to blast them into arena walls and shatter them.

### Input

**Events:** `pointerdown` / `pointerup` (both mouse and touch; no separate touch handlers needed — pointer events cover both)

**Hold phase (pointerdown held):**
- Attraction field activates immediately (0ms delay)
- All enemies within **attraction radius** are subject to magnetic force
- Force is directed toward player core center
- Force vector: `F = k / d²` where `k = 18000 units/s²·px²` and `d` = distance in pixels
  - Minimum force distance clamp: `d_min = 20px` (prevents infinite force at overlap)
  - Maximum force distance: equals attraction radius — beyond it, force = 0
- Attraction has **no maximum — enemies can reach the core**
- On core contact (enemy center within 14px of player center): enemy does NOT pass through — triggers player death (enemy has struck the core)

**Release phase (pointerup):**
- Attraction field collapses instantly
- Blast wave emanates from player center
- Blast applies **impulse** to all enemies within **blast radius**:
  - `impulse = 2200 px/s` (constant magnitude, direction = away from player center)
  - Enemies beyond blast radius: unaffected
- Blast wave visual ring: expands from 0 to `blast_radius` pixels over 220ms, then fades
- Blast is a **single instantaneous impulse** — not sustained force

### Values Table

| Parameter | Value | Notes |
|-----------|-------|-------|
| `magneticForceConstant` (k) | `18000 px³/s²` | `F = k / d²` |
| `attractionRadius` | `220px` | Level 1 default; scales per level |
| `blastImpulse` | `2200 px/s` | Instantaneous velocity addition |
| `blastRadius` | `180px` | Level 1 default; scales per level |
| `enemyMaxSpeed` | `280 px/s` | Terminal velocity cap during attraction |
| `enemyBaseSpeed` | `60 px/s` | Ambient patrol speed (Level 1) |
| `wallBounceRestitution` | `0.72` | Velocity scalar on wall bounce (28% energy loss) |
| `wallBounceMinImpulse` | `800 px/s` | Minimum post-blast wall-impact speed to kill enemy |
| `playerCoreRadius` | `12px` | Contact = death |
| `enemyRadius` | `10px` | All enemies same size |
| `deathZoneRadius` | `14px` | Player center to enemy center — collision |
| `minHoldForBlast` | `120ms` | Hold shorter than this = no blast (tap penalty) |

### Blast Kill Condition
An enemy dies when:
1. It has been blasted (within blast radius at moment of release), AND
2. Its velocity magnitude at wall contact ≥ `wallBounceMinImpulse` (800 px/s)

Enemies that bounce off walls below kill threshold: continue moving (they slow by restitution factor per bounce).

### Player Death Conditions
- Any enemy reaches within `14px` of player center (core strike)
- Three enemies survive to wall-bounce 3 times each without dying (arena overwhelm — Level 3+)

### Win Condition
- **Per level:** Kill the required enemy quota (varies by level — see Level Design)
- **Game win:** Complete all 5 levels

### Score System

| Event | Points |
|-------|--------|
| Enemy killed by wall impact | `100 × combo_multiplier` |
| Combo multiplier | +1 per kill within 1.5s window; resets after 1.5s of no kill; max 8× |
| Combo multiplier display | Shown as `×2`, `×3`… at kill site in `#FFD580` |
| Hold-time bonus | If hold ≥ 600ms before blast: `+50 pts` per enemy in blast radius |
| Perfect blast (all visible enemies hit) | `+500 pts` flat bonus |
| Level clear bonus | `1000 × level_number` |
| Enemy reaches player (death) | Score frozen; no penalty |

---

## 5. Level Design

### Level 1 — "The Crucible"
**What's new:** Tutorial-by-action. First hold. First blast. The game explains itself.
**Quota:** Kill 8 enemies to clear
**Parameters:**
- `attractionRadius`: 220px
- `blastRadius`: 180px
- `enemyCount` (simultaneous): 3
- `enemyBaseSpeed`: 60 px/s
- `enemySpawnRate`: 1 every 4.0s
- `wallBounceMinImpulse`: 700 px/s (forgiving — slightly lower kill threshold)
- **Special:** First 3 enemies spawn equidistant from player at 160px — directly in attraction radius. First hold+release guaranteed to pull and blast at least 1 within 3 seconds of game start.

---

### Level 2 — "Orbital Pressure"
**What's new:** Enemies begin orbiting at fixed radius before closing in. Player must time blast to intercept orbital path.
**Quota:** Kill 14 enemies to clear
**Parameters:**
- `attractionRadius`: 220px
- `blastRadius`: 175px
- `enemyCount`: 4
- `enemyBaseSpeed`: 80 px/s
- `enemySpawnRate`: 1 every 3.0s
- `wallBounceMinImpulse`: 800 px/s
- **New behavior:** Each enemy spawns at 280px radius and orbits (clockwise or counter-clockwise, randomized per enemy) at 80 px/s for 3–5s before beginning approach

---

### Level 3 — "Ferrous Storm"
**What's new:** Enemy count pressure. Multiple enemies attracted simultaneously create crossfire on release — they collide with each other mid-blast.
**Quota:** Kill 22 enemies to clear
**Parameters:**
- `attractionRadius`: 200px (slightly reduced — must be precise)
- `blastRadius`: 185px (slightly wider)
- `enemyCount`: 6
- `enemyBaseSpeed`: 95 px/s
- `enemySpawnRate`: 1 every 2.0s
- `wallBounceMinImpulse`: 800 px/s
- **New behavior:** Enemy-enemy collision enabled — blasted enemies can chain-kill by striking attracted-but-not-yet-blasted enemies. Chain kill = `+200 pts` bonus.
- **New threat:** "Dense" enemy variant (1 in 5 chance) — radius 14px, requires 2 wall impacts to kill; glows darker `#8B2E0A`

---

### Level 4 — "The Tether"
**What's new:** Enemies now resist attraction with partial counter-force. Hold too long and they pull *back* against the core.
**Quota:** Kill 30 enemies to clear
**Parameters:**
- `attractionRadius`: 200px
- `blastRadius`: 175px
- `enemyCount`: 6
- `enemyBaseSpeed`: 110 px/s
- `enemySpawnRate`: 1 every 1.8s
- `wallBounceMinImpulse`: 850 px/s
- **New mechanic — Counter-pull:** After being attracted for >900ms continuously, enemy activates counter-thrust: `F_resist = 8000 / d²` opposing attraction. Visible as red `#FF3300` corona on enemy. If hold exceeds 1400ms total, enemy breaks free and rockets toward player at 260 px/s.
- **New threat:** "Anchored" enemy — ignores attraction entirely; must be killed by blasted enemies as projectiles

---

### Level 5 — "Singularity"
**What's new:** The arena walls themselves pulse — contracting and expanding rhythmically at 90 BPM. Wall position oscillates ±40px. Timing blast to wall expansion = larger kill zone.
**Quota:** Kill 45 enemies to clear. Final 5 must be chain-killed in one blast.
**Parameters:**
- `attractionRadius`: 190px
- `blastRadius`: 185px
- `enemyCount`: 8
- `enemyBaseSpeed`: 130 px/s
- `enemySpawnRate`: 1 every 1.5s
- `wallBounceMinImpulse`: 900 px/s
- **Wall pulse:** Wall boundary oscillates: `wallRadius = baseWallRadius + 40 * sin(2π * (beat/4))` — synced to 90 BPM music (beat duration = 0.667s)
- **Final sequence:** Last 5 enemies spawn simultaneously in a ring at 150px — player must hold long enough to pull all 5 close, then release for a 5-kill blast. Success = game win screen. Failure (any escape) = quota resets to "kill 5 more."

---

## 6. The Moment

**Level 3, approximately 90 seconds in:**

The player has 5 enemies orbiting. They hold. Four enemies spiral inward — the attraction beams are visible as gold filaments converging on the core. The bass pitch drops. The drone rises to E5. The player waits one beat too long — one of the inner enemies flares red (counter-pull warning, even though it's only Level 3, a "dense" variant responds faster). The player releases. The blast ring explodes outward. Three enemies slam into the left wall in rapid succession — `CRACK CRACK CRACK` — the metallic synth rings ascend in pitch three times. `×3 COMBO` appears in `#FFD580` at the wall. The fourth enemy was too close — it passes *through* the blast radius already at the core boundary — and kills the player.

The player immediately restarts. They know exactly what they did wrong. They want to do it again.

---

## 7. Emotional Arc

**First 30 seconds:**
Confusion resolving into clarity. The player holds and sees the enemies *move toward them* — this is immediately legible, immediately satisfying. They release. The blast ring. The wall impact. One enemy shatters. The sound: a metallic clang. They understand the entire game in that moment. They didn't need a tutorial. The game *showed* them.

**After 2 minutes:**
Rhythm. The player is no longer thinking about the mechanic — they're *reading* the enemies. They see an orbit pattern and already know when to hold. The music has become part of their timing. When the hi-hats double during hold, it feels like confirmation of something they already knew. They are patient. They are predatory. The combo multiplier reads ×5.

**Near win (final level, last 5 enemies):**
The wall is pulsing. The arena feels alive and hostile. The 5 enemies spawn in a ring — it is unmistakably a designed moment, a gauntlet. The player holds. All 5 filaments extend toward the core. The drone reaches E5. The walls contract. The player feels the moment: *now*. They release. The world shakes 6px. Five enemies vector outward. The metallic rings ascend five times in pitch. `×5 COMBO` `×6` `×7` `×8` `PERFECT`. `+500`. The game exhales.

---

## 8. Identity in One Line

**This is the game where you hold your breath, pull the world toward you, and let go at exactly the right moment.**

---

## 9. Start Screen

### Idle Animation (game-world specific)

**Scene:** The full canvas at `#7A3E1E`. The player core sits at center — both rings rotating, core pulsing gently (scale breathes: `1.0 → 1.08 → 1.0` over 2.0s, sine easing). 

**Idle enemies:** 6 enemy orbs (`#3B1A0A` with `#FF6B35` glow) orbit the canvas at varying radii and speeds:
- Enemy 1: radius 240px, speed 0.25 rad/s, clockwise
- Enemy 2: radius 280px, speed 0.18 rad/s, counter-clockwise
- Enemy 3: radius 180px, speed 0.32 rad/s, clockwise
- Enemy 4: radius 310px, speed 0.15 rad/s, clockwise
- Enemy 5: radius 220px, speed 0.28 rad/s, counter-clockwise
- Enemy 6: radius 260px, speed 0.20 rad/s, clockwise

**Every 4 seconds:** One randomly selected idle enemy "drifts" — it smoothly arcs 40px closer to the core over 1.5s (as if testing), then retreats back over 1.0s. This suggests the game's mechanic without explaining it.

**Particle field:** 24 tiny copper dust particles (`#C45C1A`, radius 2px, alpha 0.3–0.6) drift slowly in random directions, wrapping at canvas edges. Speed: 8–18 px/s each. Randomized each page load.

**Attraction beam preview:** Every 6 seconds, 3 gold filaments (`#FFD58088`) briefly connect player core to 3 of the idle enemies — animate from core outward (duration: 0.6s) then fade (duration: 0.4s). No enemies actually move. Pure visual suggestion.

---

### SVG Overlay

#### Option A — SVG Glow Title "CORE PULL" (REQUIRED)

```svg
<svg width="800" height="800" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <!-- Outer glow: warm gold -->
    <filter id="glow-outer" x="-40%" y="-40%" width="180%" height="180%">
      <feGaussianBlur in="SourceGraphic" stdDeviation="12" result="blur-outer"/>
      <feFlood flood-color="#E8A44A" flood-opacity="0.9" result="color-outer"/>
      <feComposite in="color-outer" in2="blur-outer" operator="in" result="glow-outer"/>
      <feMerge>
        <feMergeNode in="glow-outer"/>
        <feMergeNode in="SourceGraphic"/>
      </feMerge>
    </filter>

    <!-- Inner sharp glow: bright accent -->
    <filter id="glow-inner" x="-20%" y="-20%" width="140%" height="140%">
      <feGaussianBlur in="SourceGraphic" stdDeviation="3" result="blur-inner"/>
      <feFlood flood-color="#FFD580" flood-opacity="1.0" result="color-inner"/>
      <feComposite in="color-inner" in2="blur-inner" operator="in" result="glow-inner"/>
      <feMerge>
        <feMergeNode in="glow-inner"/>
        <feMergeNode in="SourceGraphic"/>
      </feMerge>
    </filter>

    <!-- Pulsing animation: title breathes at 90 BPM (0.667s period) -->
    <!-- opacity pulses 0.85 → 1.0 → 0.85, synchronized to music feel -->
  </defs>

  <!-- Outer glow layer (wide, warm) -->
  <text
    x="400"
    y="320"
    text-anchor="middle"
    font-family="'Courier New', Courier, monospace"
    font-size="88"
    font-weight="700"
    letter-spacing="18"
    fill="#E8A44A"
    filter="url(#glow-outer)"
    opacity="0.9"
  >
    CORE
    <animate
      attributeName="opacity"
      values="0.85;1.0;0.85"
      dur="0.667s"
      repeatCount="indefinite"
    />
  </text>

  <!-- Main title: CORE -->
  <text
    x="400"
    y="320"
    text-anchor="middle"
    font-family="'Courier New', Courier, monospace"
    font-size="88"
    font-weight="700"
    letter-spacing="18"
    fill="#FFD580"
    filter="url(#glow-inner)"
  >
    CORE
  </text>

  <!-- Outer glow layer: PULL -->
  <text
    x="400"
    y="415"
    text-anchor="middle"
    font-family="'Courier New', Courier, monospace"
    font-size="88"
    font-weight="700"
    letter-spacing="18"
    fill="#E8A44A"
    filter="url(#glow-outer)"
    opacity="0.9"
  >
    PULL
    <animate
      attributeName="opacity"
      values="0.85;1.0;0.85"
      dur="0.667s"
      begin="0.333s"
      repeatCount="indefinite"
    />
  </text>

  <!-- Main title: PULL — offset phase from CORE by half period (staggered pulse) -->
  <text
    x="400"
    y="415"
    text-anchor="middle"
    font-family="'Courier New', Courier, monospace"
    font-size="88"
    font-weight="700"
    letter-spacing="18"
    fill="#FFD580"
    filter="url(#glow-inner)"
  >
    PULL
  </text>

  <!-- Separator line between CORE and PULL -->
  <line
    x1="280" y1="340" x2="520" y2="340"
    stroke="#C45C1A"
    stroke-width="1.5"
    opacity="0.6"
  />

  <!-- Tagline -->
  <text
    x="400"
    y="470"
    text-anchor="middle"
    font-family="'Courier New', Courier, monospace"
    font-size="14"
    letter-spacing="4"
    fill="#C45C1A"
    opacity="0.85"
  >
    DRAW THEM CLOSE · SHATTER THEM AGAINST THE WORLD
  </text>

  <!-- START prompt — fades in/out at 1.5s period -->
  <text
    x="400"
    y="560"
    text-anchor="middle"
    font-family="'Courier New', Courier, monospace"
    font-size="16"
    letter-spacing="6"
    fill="#E8A44A"
  >
    HOLD TO BEGIN
    <animate
      attributeName="opacity"
      values="0;1;1;0"
      keyTimes="0;0.2;0.8;1"
      dur="1.5s"
      repeatCount="indefinite"
    />
  </text>
</svg>
```

#### Option B — Mechanic Diagram Overlay

Positioned below the title (y: 580–720), a minimal SVG diagram showing:

```svg
<!-- Mechanic Diagram: Hold = attract, Release = blast -->
<svg width="800" height="160" y="620" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <filter id="diag-glow" x="-30%" y="-30%" width="160%" height="160%">
      <feGaussianBlur in="SourceGraphic" stdDeviation="4"/>
    </filter>
  </defs>

  <!-- Panel 1: HOLD -->
  <!-- Mini core -->
  <circle cx="200" cy="80" r="8" fill="#E8A44A" filter="url(#diag-glow)"/>
  <!-- 3 tiny enemies spiraling inward — static diagram, dashed paths -->
  <circle cx="160" cy="50" r="5" fill="none" stroke="#FF6B35" stroke-width="1.5"/>
  <circle cx="230" cy="40" r="5" fill="none" stroke="#FF6B35" stroke-width="1.5"/>
  <circle cx="170" cy="115" r="5" fill="none" stroke="#FF6B35" stroke-width="1.5"/>
  <!-- Arrow lines toward core -->
  <line x1="165" y1="55" x2="195" y2="74" stroke="#FFD580" stroke-width="1" stroke-dasharray="3,2" opacity="0.7"/>
  <line x1="225" y1="47" x2="205" y2="72" stroke="#FFD580" stroke-width="1" stroke-dasharray="3,2" opacity="0.7"/>
  <line x1="175" y1="110" x2="196" y2="88" stroke="#FFD580" stroke-width="1" stroke-dasharray="3,2" opacity="0.7"/>
  <!-- Label -->
  <text x="200" y="138" text-anchor="middle" font-family="'Courier New', monospace" font-size="10" letter-spacing="3" fill="#C45C1A">HOLD</text>

  <!-- Divider -->
  <line x1="300" y1="30" x2="300" y2="130" stroke="#5C2A0E" stroke-width="1" opacity="0.5"/>
  <text x="350" y="85" text-anchor="middle" font-family="'Courier New', monospace" font-size="11" letter-spacing="2" fill="#7A3E1E" opacity="0">·</text>

  <!-- Panel 2: RELEASE -->
  <circle cx="540" cy="80" r="8" fill="#E8A44A" filter="url(#diag-glow)"/>
  <!-- Blast ring -->
  <circle cx="540" cy="80" r="40" fill="none" stroke="#FFD580" stroke-width="1.5" opacity="0.5" stroke-dasharray="4,3"/>
  <!-- 3 enemies blasting outward -->
  <circle cx="490" cy="40" r="5" fill="none" stroke="#FF6B35" stroke-width="1.5"/>
  <circle cx="590" cy="45" r="5" fill="none" stroke="#FF6B35" stroke-width="1.5"/>
  <circle cx="510" cy="122" r="5" fill="none" stroke="#FF6B35" stroke-width="1.5"/>
  <!-- Arrow lines away from core -->
  <line x1="495" y1="48" x2="505" y2="62" stroke="#FFD580" stroke-width="1" opacity="0.7"/>
  <line x1="583" y1="52" x2="572" y2="64" stroke="#FFD580" stroke-width="1" opacity="0.7"/>
  <line x1="516" y1="115" x2="523" y2="102" stroke="#FFD580" stroke-width="1" opacity="0.7"/>
  <!-- Label -->
  <text x="540" y="138" text-anchor="middle" font-family="'Courier New', monospace" font-size="10" letter-spacing="3" fill="#C45C1A">RELEASE</text>
</svg>
```

---

### Start Screen Timing Summary

| Event | Timing |
|-------|--------|
| Canvas background fill | Immediate |
| Idle enemies begin orbiting | Immediate |
| Particle field starts | Immediate |
| SVG overlay fades in | 0 → 1 alpha over 800ms |
| "HOLD TO BEGIN" blink starts | After 1.2s |
| Attraction beam preview pulses | Every 6s, starting at 3s |
| Enemy "drift and retreat" | Every 4s, starting at 2s |
| Music begins | On first `pointerdown` to start |

---

## Implementation Notes for Programmer

- All physics in world-space pixels; canvas is 800×800 logical, scaled via CSS `transform: scale()`
- `pointerdown` / `pointerup` events on `document` (not canvas element) — ensures mobile touch works regardless of tap location
- `requestAnimationFrame` loop; delta-time capped at `dt_max = 0.033s` (30fps floor)
- Enemy positions initialized outside player `attractionRadius` (≥ 240px from center)
- Blast impulse applied as: `enemy.velocity += normalize(enemy.pos - player.pos) * blastImpulse`
- Wall collision: if `|enemy.pos - center| + enemy.radius > arenaRadius`, reflect velocity component and apply `wallBounceRestitution = 0.72`
- Check kill condition at wall bounce: `if velocity.magnitude ≥ wallBounceMinImpulse → enemy.alive = false`
- Attraction field visual: draw `N` line segments from player center to each attracted enemy, alpha proportional to `1 - (d / attractionRadius)`, color `#FFD58088`
- Bloom: render game to offscreen canvas, apply Gaussian blur (radius 18px), blend additively at 0.6 opacity onto main canvas

---

*Round 17 — Core Pull Design Document*
*Every value is intentional. Every frame is earned.*
