# Lunar Lander — Game Design Document
**Version:** 1.1  
**Date:** February 2026  
**Platform:** Browser (HTML/JS)

---

## 1. Overview

Lunar Lander is a realistic, tension-driven spaceflight simulation game in the tradition of the Apollo program. The player pilots a descent module through increasingly hostile environments — from the barren Moon to the violent storms of Jupiter's moons — with one goal: land safely, or die trying.

The game rewards precision, patience, and fuel conservation. Every mission feels like a genuine test of skill. There are no shortcuts.

---

## 2. Core Concept

**Genre:** Physics simulation / skill game  
**Tone:** Realistic & tense — Apollo-style sim  
**Perspective:** 2D side-view  
**Session length:** 3–10 minutes per level  

The player controls a lander using rotation and main engine thrust. Gravity pulls the ship down. Fuel is finite. The terrain is treacherous. A successful landing requires near-perfect vertical alignment, low descent velocity, and precise touchdown on the designated pad.

---

## 3. Player Controls

| Input | Action |
|-------|--------|
| Left Arrow / A | Rotate lander counter-clockwise |
| Right Arrow / D | Rotate lander clockwise |
| Up Arrow / W / Space | Fire main thruster |

There are no lateral thrusters. The player must rotate the lander to redirect thrust. This is the core skill mechanic — managing orientation and thrust timing simultaneously.

---

## 4. Core Mechanics

### 4.1 Physics
- Gravity is applied continuously as a downward acceleration force
- Thrust is applied in the direction the lander is currently facing
- The lander has mass, and both gravity and thrust affect velocity over time
- Rotation has a fixed angular velocity (no angular inertia)
- The lander can drift horizontally as well as vertically

### 4.2 Fuel
- The lander starts each level with a set amount of fuel
- Every frame the thruster fires, fuel is consumed
- When fuel reaches zero, thrust is disabled — only gravity remains
- Fuel remaining at landing contributes to the star rating
- There is no fuel refill mid-level

### 4.3 Landing Detection
A successful landing requires ALL of the following to be true at the moment of touchdown:

- The lander is touching a designated landing pad
- Vertical velocity is below the **safe descent speed threshold** (varies by level)
- The lander's angle is within **±5 degrees** of vertical
- Horizontal velocity is below a safe threshold

If any condition fails, the landing is a crash.

### 4.4 Crash Conditions
A crash occurs if:
- The lander contacts terrain at unsafe speed or angle
- The lander contacts terrain outside a landing pad
- The lander flips upside down and contacts anything

### 4.5 Lives System
- The player begins with **3 lives**
- Each crash costs one life
- Losing all lives triggers **Game Over** and returns to the level select screen
- Lives do not replenish between levels (across a single run)
- A future upgrade system may allow extra lives as rewards

---

## 5. Win / Loss Conditions

**Level Win:** Successfully land on the designated pad meeting all landing criteria  
**Level Fail (life lost):** Crash — lose one life, restart the current level  
**Game Over:** All lives lost — return to level select  
**Campaign Win:** Complete all 10 levels with at least 1 life remaining  

---

## 6. Star Rating System

Each level awards 1–3 stars based on performance:

| Stars | Conditions |
|-------|-----------|
| ⭐ | Landed successfully |
| ⭐⭐ | Landed with >40% fuel remaining AND descent speed < 60% of threshold |
| ⭐⭐⭐ | Landed with >65% fuel remaining AND near-perfect touchdown (speed < 30% of threshold, angle < 2°) |

Stars are recorded per level. The player can replay levels to improve their star rating. Stars are displayed on the level select screen.

---

## 7. Level Progression

The game contains **12 levels** across **4 planetary environments** (3 levels each). Difficulty increases through a combination of:

- Narrower landing pads
- Stronger gravity
- Less starting fuel
- Wind / atmospheric drag (from Level 5 onward)
- Debris and obstacles (from Level 7 onward)
- Multiple possible landing pads (only one is valid — indicated by a beacon)

### 7.1 Environments

| Environment | Levels | Gravity | Hazards |
|-------------|--------|---------|---------|
| The Moon | 1–3 | Low (1.62 m/s²) | None |
| Mars | 4–6 | Medium (3.72 m/s²) | Dust storms (wind) |
| Titan (Saturn's moon) | 7–9 | Low-medium (1.35 m/s²) | Dense atmosphere (drag), debris fields, **UFO** |
| Io (Jupiter's moon) | 10–12 | Medium-high (1.80 m/s²) | Wind + debris + narrow pads, **UFO** |

### 7.2 Level Unlock
Levels unlock sequentially. The player must complete a level (1 star minimum) to unlock the next.

---

## 8. Hazards

### 8.1 Wind / Atmospheric Drag
- Applies a horizontal force to the lander
- Wind direction and strength vary by level
- Wind is visible via particle effects (dust, atmospheric shimmer)
- Wind can gust — sudden increases shown by a brief UI indicator

### 8.2 Debris & Obstacles
- Static or slow-moving rocks/debris float in the descent path
- Collision with debris = instant crash (same as terrain collision)
- Debris is clearly visible and avoidable with skill
- Some levels have debris fields the player must navigate through

### 8.3 The UFO
From Level 7 onward, a single UFO appears in the airspace around the lander. It is not hostile — it is **curious**. It has noticed the lander and wants to observe it. This creates passive jeopardy: the UFO drifts through the same airspace the player must navigate, becoming an unpredictable obstacle.

**Behaviour:**
- The UFO orbits loosely around the lander using smooth sinusoidal drift patterns
- It maintains a general proximity to the lander but does not track it precisely — it lags, overshoots, and wanders
- The UFO does not react to the player's inputs or attempt to intercept the lander
- It never blocks the landing pad intentionally — it roams the mid-to-upper airspace
- On levels 10–12 (Io) the UFO moves slightly faster, reflecting increased alien curiosity

**Collision:**
- If the lander contacts the UFO, the lander is **damaged** — a percentage of remaining fuel is instantly drained (25% of current fuel per collision)
- The UFO is unaffected — it continues orbiting as if nothing happened
- Multiple collisions in a single run are possible and increasingly costly
- There is no warning before collision; the player must track the UFO visually

**Visual:**
- Classic saucer shape — flat disc with a domed top
- Soft pulsing glow (cool blue/white) to make it readable against all backgrounds
- Small light trail to help the player anticipate its movement direction
- Slightly transparent — clearly alien, slightly unreal

**Audio intent:**
- Subtle electronic hum when UFO is within close proximity
- Brief dissonant tone on collision

---

## 9. Landing Success Animations

Each successful landing triggers a short cinematic celebration sequence unique to the planet environment. The camera zooms out to reveal the full scene, the animation plays, then the screen fades to the results screen.

### Animation Structure (all levels)
1. **Landing confirmed** — lander settles, legs compress slightly, dust puff at contact point
2. **Hatch opens** — top hatch swings open (0.5s)
3. **Astronaut emerges** — climbs out onto the surface
4. **Astronaut walks** — moves to flag planting spot using planet-appropriate gait
5. **Flag planted** — dust puff, flag unfurls
6. **Hold** — 2 second hold on the completed scene
7. **Fade to results screen** — smooth fade transition

### Camera Behaviour
- On landing confirmation, camera begins a slow cinematic zoom out
- Zoom reveals the full lander, surrounding terrain, and sky
- Camera holds on the wide shot for the duration of the animation
- No player input accepted during the sequence (inputs are consumed silently)
- A subtle letterbox (black bars top/bottom) activates to reinforce the cinematic feel

### Per-Planet Animations

#### The Moon (Levels 1–3)
- Astronaut walks with a **bouncy low-gravity gait** — each step floats slightly before landing
- Carries and plants an **Earth flag** (or mission flag)
- Small **grey dust puff** on flag plant
- Scene held: astronaut stands at attention beside the flag, lander behind them
- Tone: historic, proud, quiet

#### Mars (Levels 4–6)
- Astronaut walks in a **pressurised suit with heavier steps** — Mars gravity is more noticeable
- Plants a **mission beacon** rather than a flag (a blinking transmitter spike)
- **Red dust cloud** billows on planting, drifts sideways in the Martian wind
- Astronaut shields visor with one arm against the dust
- Tone: industrial, frontier, harsh

#### Titan (Levels 7–9)
- Astronaut walks in a **thick insulated suit**, slow and deliberate
- Plants a **scientific sensor array** — a small tripod with antennae
- **Golden/amber vapour puff** on planting (Titan's thick atmosphere)
- If the UFO is present, it hovers curiously closer to observe the planted equipment — then drifts away
- Tone: eerie, mysterious, alien

#### Io (Levels 10–12)
- Astronaut moves **quickly and urgently** — Io is volcanically active and hostile
- Plants a **hardened black monolith** marker (sleek, futuristic)
- Small **sulphurous yellow dust puff** on planting, with a faint tremor ripple across the ground
- Astronaut immediately turns and jogs back to the lander — no lingering
- Tone: dangerous, urgent, triumphant

### Skippable
- Player may press **Escape or Space** to skip the animation and go directly to the results screen
- A small "Press Space to skip" hint appears in the bottom-right corner after 1 second

---

## 10. Visual Style

**Style:** Clean modern flat design  
**Palette:** Dark space backgrounds, planet-appropriate terrain colours, bright accent colours for UI and landing pads  
**Lander:** Simple geometric shape — triangular body, landing legs, thruster cone  
**Thruster effect:** Animated flame particle system (subtle but readable)  
**Terrain:** Smooth polygonal silhouettes — no pixel art  
**HUD:** Minimal — velocity, fuel bar, altitude, angle indicator, lives  
**Animations:** Smooth 60fps — lander rotation, thrust particles, crash explosion, landing success flourish  

### Colour Palette Reference

| Element | Colour |
|---------|--------|
| Background (space) | #0A0E1A |
| Moon terrain | #C8C8C8 |
| Mars terrain | #B5451B |
| Titan terrain | #D4A843 |
| Io terrain | #E8C840 |
| Landing pad | #00FF88 |
| Landing pad (locked) | #FF4444 |
| Lander | #E8E8E8 |
| Thrust flame | #FF8C00 → #FFFF00 |
| HUD text | #FFFFFF |
| Fuel bar (safe) | #00FF88 |
| UFO body | #C8E8FF |
| UFO glow | #7AB8FF (pulsing) |

---

## 11. HUD / UI

### In-Level HUD
- **Fuel bar** — horizontal bar, top-left, changes colour when low (<25%)
- **Velocity indicator** — shows vertical and horizontal speed in m/s
- **Altitude** — distance from lander to nearest terrain below
- **Angle indicator** — shows current lander tilt in degrees
- **Lives** — small lander icons, top-right
- **Level name** — top-centre, fades after 3 seconds

### Menus
- **Main Menu:** Title, Play, How to Play, Credits
- **Level Select:** Grid of 12 levels, star ratings shown, locked levels greyed
- **Pause Menu:** Resume, Restart, Level Select, Main Menu
- **Level Complete:** Stars awarded, fuel remaining shown, Next Level button
- **Game Over:** Lives lost message, Retry / Level Select buttons

---

## 12. Audio (Design Intent)

*Audio will be designed in a separate spec. Placeholder notes:*

- Ambient space silence with subtle low-frequency hum
- Thruster: reactive burn sound, pitch shifts with fuel level
- Wind: atmospheric howl on affected levels
- Crash: short sharp impact sound + silence
- Landing success: satisfying thud + short musical sting
- Success animation: short triumphant musical sting, planet-appropriate ambient sound
- UFO proximity: soft electronic hum that grows louder as UFO approaches
- UFO collision: brief dissonant tone + fuel drain visual flash

---

## 13. Scoring & Persistence

- Star ratings per level are saved to localStorage
- Best fuel-remaining percentage per level is saved
- Total stars across all levels shown on main menu
- No online leaderboard in v1.0

---

## 14. Spec Files (Planned)

The following spec files will be written from this GDD:

1. **SPEC-01-Physics** — gravity, thrust, drag, rotation, collision
2. **SPEC-02-Controls** — input handling, key bindings, responsiveness
3. **SPEC-03-Fuel** — fuel consumption, display, empty state
4. **SPEC-04-LandingDetection** — success/fail criteria, tolerances
5. **SPEC-05-LivesAndGameOver** — life tracking, game over flow
6. **SPEC-06-Terrain** — terrain generation, per-level definitions
7. **SPEC-07-Hazards** — wind, debris, per-level configuration
8. **SPEC-08-UFO** — UFO behaviour, movement, collision, visual/audio
9. **SPEC-09-SuccessAnimation** — per-planet celebration sequences, camera, transitions
10. **SPEC-10-ScoringAndStars** — rating calculation, persistence
11. **SPEC-11-HUDandUI** — all on-screen elements, menus, transitions
12. **SPEC-12-LevelDefinitions** — all 12 levels, parameters, layouts

---

## 15. Out of Scope (v1.0)

- Mobile / touch controls
- Multiplayer
- Procedural terrain generation
- Online leaderboards
- Sound design (placeholder only)
- Story / narrative cutscenes
