# SPEC-07 — Hazards
**Game:** Lunar Lander  
**Version:** 1.0  
**Status:** Draft  
**Depends on:** GDD v1.2, SPEC-01-Physics, SPEC-04-LandingDetection, SPEC-06-Terrain  
**Used by:** SPEC-12-LevelDefinitions

---

## 1. Purpose

This spec defines the two environmental hazards — **wind** and **debris** — that make levels progressively more dangerous from Level 4 onward. The UFO is a separate entity with its own spec (SPEC-08). Wind and atmospheric drag are physically simulated in SPEC-01; this spec defines their per-level parameters, visual representation, and the player-facing feedback systems (wind indicator, gust warning). Debris is defined here in full — its data model, collision behaviour, visual style, and per-level configuration.

---

## 2. Hazard Activation by Level

| Level | Environment | Wind | Atmospheric Drag | Debris | UFO |
|-------|-------------|------|-----------------|--------|-----|
| 1–3   | Moon        | No   | No              | No     | No  |
| 4     | Mars        | Yes  | Yes (subtle)    | No     | No  |
| 5     | Mars        | Yes  | Yes (subtle)    | No     | No  |
| 6     | Mars        | Yes  | Yes (subtle)    | No     | No  |
| 7     | Titan       | No   | Yes (strong)    | Yes    | Yes |
| 8     | Titan       | No   | Yes (strong)    | Yes    | Yes |
| 9     | Titan       | No   | Yes (strong)    | Yes    | Yes |
| 10    | Io          | Yes  | Yes (minimal)   | Yes    | Yes |
| 11    | Io          | Yes  | Yes (minimal)   | Yes    | Yes |
| 12    | Io          | Yes  | Yes (minimal)   | Yes    | Yes |

---

## 3. Wind

Wind physics are implemented in SPEC-01 §10. This section defines the per-level wind parameters and all player-facing wind feedback.

### 3.1 Wind Parameters

Each level with wind has the following parameters defined in SPEC-12:

```javascript
level.wind          // base horizontal acceleration in m/s² — positive = right, negative = left
level.gustEnabled   // boolean — whether gusts occur
level.gustAmplitude // m/s² — max additional force during a gust
level.gustFrequency // Hz — how fast gusts oscillate (drives the sin function in SPEC-01 §10)
```

Reference values (confirmed in SPEC-12):

| Level | `wind` (m/s²) | `gustEnabled` | `gustAmplitude` | `gustFrequency` |
|-------|--------------|---------------|-----------------|-----------------|
| 4     | +0.8         | false         | —               | —               |
| 5     | -1.2         | true          | 0.6             | 0.4             |
| 6     | +1.5         | true          | 1.0             | 0.5             |
| 10    | -2.0         | true          | 1.5             | 0.6             |
| 11    | +2.5         | true          | 2.0             | 0.7             |
| 12    | -3.0         | true          | 2.5             | 0.8             |

Wind direction alternates between levels (positive/negative) to prevent the player developing a single-direction compensation habit.

### 3.2 Wind Visual Feedback

Wind is not directly visible — the player infers it from the lander's behaviour. To make it readable, two visual systems communicate wind state:

**Particle streams:** Horizontal particle trails flow across the canvas in the wind direction. Particle count and speed scale with wind strength.

```javascript
// Wind particle system — updated each frame
const WIND_PARTICLES = {
  count:    30,              // active particles at once
  speed:    level.wind * 40, // pixels per second (scales with wind)
  y:        'random',        // spawn at random Y positions
  opacity:  0.3,             // subtle — not distracting
  length:   12,              // particle trail length in pixels
  colour:   level.windParticleColour,
};
```

Per-environment wind particle colours:

| Environment | Particle Colour | Appearance |
|-------------|----------------|------------|
| Mars        | `#D4622A`      | Rust-coloured dust motes |
| Io          | `#E8C840`      | Sulphurous yellow wisps |

**Wind direction indicator:** A small arrow icon in the HUD shows current wind direction and a qualitative strength reading (Low / Medium / High). Defined in SPEC-11.

### 3.3 Gust Warning

When a gust is active (wind force exceeds base wind by more than 50% of `gustAmplitude`), a brief HUD warning appears:

```javascript
const currentGust = Math.abs(totalWind) - Math.abs(level.wind);
const gustThreshold = level.gustAmplitude * 0.5;

if (level.gustEnabled && currentGust > gustThreshold) {
  hud.showGustWarning(); // flashes "GUST" text for 0.4s
}
```

The gust warning is cosmetic — it does not pause or slow the game. It gives attentive players a fraction of a second to react.

---

## 4. Atmospheric Drag

Drag physics are implemented in SPEC-01 §9. Drag coefficients are defined per environment and assigned from SPEC-12. No additional player feedback for drag — it is a background force the player adapts to over time.

| Environment | `level.drag` | Perceptible Effect |
|-------------|-------------|-------------------|
| Moon        | 0.0000      | None |
| Mars        | 0.0002      | Subtle — slight terminal velocity reduction |
| Titan       | 0.0008      | Noticeable — lander feels sluggish at speed |
| Io          | 0.0001      | Minimal — barely perceptible |

Drag has no visual representation. Its effect is felt through the lander's handling. On Titan, experienced players will notice the lander decelerates slightly when not thrusting — this is intentional and contributes to Titan's eerie, heavy-atmosphere feel.

---

## 5. Debris

Debris are solid obstacles positioned in the airspace of levels 7–12. Collision with any debris piece is an immediate crash — same consequence as hitting terrain.

### 5.1 Debris Data Model

Each debris piece is a static or slowly drifting object defined per level:

```javascript
// Single debris piece definition
{
  id:       'debris_01',
  shape:    'rock',         // 'rock' | 'boulder' | 'shard' — affects visual only
  x:        420,            // canvas X position (centre)
  y:        280,            // canvas Y position (centre)
  width:    30,             // collision bounding box width
  height:   24,             // collision bounding box height
  drift:    0.0,            // horizontal drift speed in px/s (0 = static)
  driftMin: 0,              // left drift boundary (canvas X) — ignored if drift = 0
  driftMax: 0,              // right drift boundary (canvas X) — ignored if drift = 0
  rotation: 14,             // visual rotation in degrees (cosmetic only)
}
```

### 5.2 Debris Movement

Debris with `drift !== 0` oscillates slowly between `driftMin` and `driftMax`. The movement uses a simple ping-pong:

```javascript
function updateDebris(piece, dt) {
  if (piece.drift === 0) return;

  piece.x += piece.driftDirection * piece.drift * dt;

  if (piece.x >= piece.driftMax) {
    piece.x = piece.driftMax;
    piece.driftDirection = -1;
  }
  if (piece.x <= piece.driftMin) {
    piece.x = piece.driftMin;
    piece.driftDirection = 1;
  }
}
```

`piece.driftDirection` is initialised to `1` (rightward) at level start. Drifting debris is slow enough (max 20 px/s) that it is always avoidable with skill.

### 5.3 Debris Collision Detection

Debris collision uses **axis-aligned bounding box (AABB)** detection against the lander's bounding box. This is a deliberate simplification — debris shapes are irregular, but AABB collision is fast, predictable, and fair.

```javascript
function checkDebrisCollision(lander, piece) {
  const landerLeft   = lander.x - LANDER.width / 2;
  const landerRight  = lander.x + LANDER.width / 2;
  const landerTop    = lander.y - LANDER.height / 2;
  const landerBottom = lander.y + LANDER.height / 2;

  const debrisLeft   = piece.x - piece.width / 2;
  const debrisRight  = piece.x + piece.width / 2;
  const debrisTop    = piece.y - piece.height / 2;
  const debrisBottom = piece.y + piece.height / 2;

  const overlapping =
    landerRight  > debrisLeft  &&
    landerLeft   < debrisRight &&
    landerBottom > debrisTop   &&
    landerTop    < debrisBottom;

  if (overlapping) {
    triggerCrash('TERRAIN_CONTACT'); // debris collision = same as terrain crash
  }
}
```

Debris collision is checked in the physics loop after terrain collision (SPEC-01 §3, step 9). If terrain collision is already detected on the same frame, debris collision is skipped — the crash is already triggering.

### 5.4 Debris Visual Style

Debris are rendered as irregular polygon shapes filled with the planet's terrain colour, slightly darker.

```javascript
// Debris fill colours (slightly darker than terrain)
const DEBRIS_COLOURS = {
  TITAN: '#A07830',   // darker than #D4A843 terrain
  IO:    '#C4A430',   // darker than #E8C840 terrain
};
```

Each debris `shape` type maps to a predefined polygon outline:

| Shape | Description | Approx. Size |
|-------|-------------|-------------|
| `rock` | Rounded irregular blob | 20–35px |
| `boulder` | Large angular chunk | 35–55px |
| `shard` | Thin elongated spike | 10×40px |

Shards are especially dangerous — narrow but long, they are easy to misjudge when descending quickly.

### 5.5 Debris Placement Rules

Applied when authoring debris positions in SPEC-12:

| Rule | Requirement |
|------|-------------|
| Minimum clearance | At least 80px between any debris piece and the active landing pad |
| Avoidable | Every debris piece must have a clear flight path around it of at least 60px width |
| No spawn overlap | No debris within 150px below the lander spawn point |
| Terrain clearance | Debris must not overlap or sit on terrain — minimum 20px above terrain surface |
| Horizontal wrap | Debris positioned within 60px of either canvas edge must account for lander wrap |

### 5.6 Debris Count by Level

| Level | Environment | Static Debris | Drifting Debris |
|-------|-------------|--------------|-----------------|
| 7     | Titan       | 2            | 1               |
| 8     | Titan       | 3            | 2               |
| 9     | Titan       | 4            | 2               |
| 10    | Io          | 3            | 2               |
| 11    | Io          | 4            | 3               |
| 12    | Io          | 5            | 3               |

On Io, debris and wind combine. Wind pushes the lander sideways while debris occupies the lateral space the player might use to correct. This is the intended design tension for the final three levels.

---

## 6. Hazard Interaction Summary

Hazards do not interact with each other, but they interact with the lander simultaneously. The physics loop processes them in this order each frame:

1. Wind force applied (SPEC-01 §10)
2. Drag applied (SPEC-01 §9)
3. Terrain collision check (SPEC-01 §11)
4. Debris collision check (this spec §5.3)
5. UFO collision check (SPEC-08)
6. Landing pad collision check (SPEC-04)

Multiple hazards can affect the lander in the same frame. A player can be pushed by a gust into a debris piece and crash — this is fair because both hazards are visible and the crash is avoidable with skill.

---

## 7. Hazard Constants Summary

```javascript
const HAZARDS = {
  WIND: {
    GUST_WARN_THRESHOLD: 0.5,    // fraction of gustAmplitude that triggers HUD warning
    GUST_WARN_DURATION:  400,    // ms the "GUST" HUD text is displayed
    PARTICLE_COUNT:      30,
    PARTICLE_OPACITY:    0.3,
    PARTICLE_LENGTH:     12,     // pixels
  },
  DEBRIS: {
    MAX_DRIFT_SPEED:     20,     // px/s — max speed of drifting debris
    MIN_PAD_CLEARANCE:   80,     // px from any debris to active landing pad
    MIN_FLIGHT_PATH:     60,     // px clear width around each debris piece
    SPAWN_CLEARANCE:     150,    // px below spawn point kept debris-free
    TERRAIN_CLEARANCE:   20,     // px above terrain surface minimum
  },
};
```

---

## 8. Resolved Decisions

| # | Question | Decision | Rationale |
|---|----------|----------|-----------|
| 1 | Should wind be visible directly or only via particles? | Particles + HUD arrow only | Direct wind visibility (e.g. visible force arrows on the lander) would be cluttered and unrealistic. Particles give environmental flavour; the HUD arrow gives clear directional info. The player learns to read wind through lander behaviour — this is a skill the game rewards. |
| 2 | Should debris use pixel-perfect or AABB collision? | AABB — axis-aligned bounding box | Pixel-perfect collision is expensive and produces unexpected results near debris edges. AABB is slightly generous but fast and predictable. Players understand rectangular hitboxes intuitively. |
| 3 | Should drifting debris move randomly or predictably? | Predictably — ping-pong between set bounds | Random movement would feel unfair. Predictable ping-pong means an attentive player can learn debris timing and plan around it. Skill-based avoidance, not luck. |
| 4 | Should debris exist on Mars levels? | No — debris starts on Titan (Level 7) | Mars already introduces wind. Adding debris on Mars too would make the difficulty jump too steep too early. Titan gets debris as its primary challenge; wind returns on Io with debris already established. |

---

*End of SPEC-07-Hazards*
