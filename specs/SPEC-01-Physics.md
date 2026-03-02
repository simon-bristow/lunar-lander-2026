# SPEC-01 — Physics
**Game:** Lunar Lander  
**Version:** 1.0  
**Status:** Draft  
**Depends on:** GDD v1.2  
**Used by:** SPEC-02-Controls, SPEC-03-Fuel, SPEC-04-LandingDetection, SPEC-07-Hazards, SPEC-08-UFO

---

## 1. Purpose

This spec defines the complete physics simulation for the lander. It covers the game loop update order, coordinate system, gravity, thrust, rotation, atmospheric drag, collision geometry, and all per-planet constants. Any system that moves, accelerates, or collides with the lander reads from this spec.

---

## 2. Coordinate System

- **Origin (0, 0):** top-left of the canvas
- **X axis:** positive = right
- **Y axis:** positive = down (standard canvas convention)
- **Angle:** measured in degrees, clockwise from straight up (0° = lander pointing up, 90° = pointing right, 180° = pointing down, 270° = pointing left)
- **Velocity:** stored as `vx` (horizontal) and `vy` (vertical) in pixels per second
- **All physics constants** are defined in real-world units (m/s²) then scaled to pixels via `PIXELS_PER_METRE = 6`

```
PIXELS_PER_METRE = 6
// 1 metre = 6 pixels at base canvas resolution (900×600)
// All gravity and thrust values multiply by this scale factor
```

---

## 3. Game Loop & Update Order

The physics engine runs inside `requestAnimationFrame`. Each frame:

```
1. Compute deltaTime (dt) in seconds — capped at 0.05s (see §3.1)
2. Apply rotation input
3. Apply thrust (if firing AND fuel > 0)
4. Apply gravity
5. Apply atmospheric drag (if applicable to current level)
6. Apply wind force (if applicable to current level)
7. Update position from velocity
8. Check terrain collision
9. Check debris collision
10. Check UFO collision
11. Check landing pad collision
```

Order is strict. Rotation must update angle before thrust is computed so thrust fires in the correct direction.

### 3.1 Delta Time Cap

```javascript
const MAX_DT = 0.05; // seconds (= 20fps minimum)
dt = Math.min(rawDt, MAX_DT);
```

Capping dt prevents physics tunnelling when the tab is backgrounded or the frame rate drops severely. At 60fps, dt ≈ 0.0167s. At 30fps, dt ≈ 0.033s. Both are well within the cap.

---

## 4. Lander Properties

These are fixed constants for the lander object. They do not change between levels.

```javascript
const LANDER = {
  mass:            1.0,    // kg (normalised — used for force application)
  width:           40,     // pixels (bounding box width)
  height:          50,     // pixels (bounding box height)
  legSpan:         52,     // pixels (distance between leg tips — used for landing detection)
  thrustPower:     18.0,   // m/s² acceleration when thruster fires
  rotationSpeed:   120,    // degrees per second (fixed angular velocity, no inertia)
};
```

### 4.1 Lander Collision Shape

The lander uses a **simplified collision polygon** — not the visual sprite outline. This keeps collision consistent and fair.

```
Collision shape (relative to lander centre, angle = 0):
  Top point:    (0, -22)
  Right point:  (+18, +10)
  Bottom-right: (+20, +22)   ← leg tip
  Bottom-left:  (-20, +22)   ← leg tip
  Left point:   (-18, +10)

This is a 5-point convex polygon.
When the lander rotates, all points are rotated by the lander's current angle.
```

For collision detection purposes, the two **leg tip points** are the primary contact points for landing detection (see SPEC-04). All five points are used for terrain and debris collision (crash detection).

---

## 5. Gravity

Gravity is a constant downward acceleration applied every frame.

```javascript
// Applied each frame:
lander.vy += gravity * PIXELS_PER_METRE * dt;
```

Gravity values are defined per planet environment. The value in use is set by the level definition (see SPEC-12).

| Environment | Levels | Real Gravity (m/s²) | Scaled (px/s²) |
|-------------|--------|---------------------|----------------|
| The Moon    | 1–3    | 1.62                | 9.72           |
| Mars        | 4–6    | 3.72                | 22.32          |
| Titan       | 7–9    | 1.35                | 8.10           |
| Io          | 10–12  | 1.80                | 10.80          |

```javascript
// Example: Moon
const GRAVITY_MOON = 1.62; // m/s²
lander.vy += GRAVITY_MOON * PIXELS_PER_METRE * dt;
```

---

## 6. Thrust

Thrust is applied when the player holds the thrust key AND `lander.fuel > 0`.

Thrust accelerates the lander in the direction it is currently facing (determined by `lander.angle`).

```javascript
if (input.thrust && lander.fuel > 0) {
  const angleRad = (lander.angle - 90) * (Math.PI / 180);
  // Subtract 90° because angle 0 = pointing up, but cos/sin use right as 0
  const ax = Math.cos(angleRad) * LANDER.thrustPower * PIXELS_PER_METRE;
  const ay = Math.sin(angleRad) * LANDER.thrustPower * PIXELS_PER_METRE;
  lander.vx += ax * dt;
  lander.vy += ay * dt;
  // Fuel consumption handled in SPEC-03
}
```

### 6.1 Thrust Visual Feedback

When thrust fires, a flame particle system activates at the thruster nozzle (bottom of lander when angle = 0). The nozzle position in local space is `(0, +28)`, rotated by lander angle into world space.

---

## 7. Rotation

Rotation is a fixed angular velocity — there is no rotational inertia. The lander turns at a constant rate while the rotate key is held.

```javascript
if (input.rotateLeft) {
  lander.angle -= LANDER.rotationSpeed * dt;
}
if (input.rotateRight) {
  lander.angle += LANDER.rotationSpeed * dt;
}

// Normalise angle to [0, 360)
lander.angle = ((lander.angle % 360) + 360) % 360;
```

Rotation and thrust may be applied simultaneously. This is intentional — the player can rotate while burning to steer.

---

## 8. Position Update

After all forces are applied, update position from velocity:

```javascript
lander.x += lander.vx * dt;
lander.y += lander.vy * dt;
```

### 8.1 Canvas Boundary Behaviour

- **Left / Right edges:** lander wraps to the opposite side (seamless horizontal wrapping)
- **Top edge:** lander is clamped — it cannot leave the top of the screen; `vy` is set to 0 if the lander reaches y = 0
- **Bottom edge:** handled by terrain collision — the lander should never reach the raw canvas bottom without hitting terrain first

```javascript
// Horizontal wrap
if (lander.x > CANVAS_WIDTH + LANDER.width / 2) lander.x = -LANDER.width / 2;
if (lander.x < -LANDER.width / 2) lander.x = CANVAS_WIDTH + LANDER.width / 2;

// Top clamp
if (lander.y < 0) {
  lander.y = 0;
  lander.vy = 0;
}
```

---

## 9. Atmospheric Drag

Drag is only active on planets with atmospheres. It is **not** present on the Moon.

Drag applies a force opposing current velocity, proportional to speed squared (simplified quadratic drag).

```javascript
if (level.drag > 0) {
  const speed = Math.sqrt(lander.vx ** 2 + lander.vy ** 2);
  if (speed > 0) {
    const dragForce = level.drag * speed * speed;
    const dragX = -(lander.vx / speed) * dragForce * dt;
    const dragY = -(lander.vy / speed) * dragForce * dt;
    lander.vx += dragX;
    lander.vy += dragY;
  }
}
```

Drag coefficients per environment:

| Environment | `level.drag` | Effect |
|-------------|-------------|--------|
| Moon        | 0.0000      | None   |
| Mars        | 0.0002      | Subtle — barely noticeable, slight terminal velocity |
| Titan       | 0.0008      | Noticeable — Titan has a thick nitrogen atmosphere |
| Io          | 0.0001      | Minimal — Io has a very thin SO₂ atmosphere |

---

## 10. Wind

Wind applies a constant (with optional gust) horizontal acceleration force. Active from Level 5 onward.

```javascript
if (level.wind !== 0) {
  const gustOffset = level.gustEnabled
    ? Math.sin(Date.now() * level.gustFrequency) * level.gustAmplitude
    : 0;
  const totalWind = level.wind + gustOffset;
  lander.vx += totalWind * PIXELS_PER_METRE * dt;
}
```

Wind parameters are defined per level in SPEC-12. Example values:

| Level | `level.wind` (m/s²) | `level.gustEnabled` | `level.gustAmplitude` |
|-------|---------------------|---------------------|----------------------|
| 4 (Mars)  | 0.8  | false | —   |
| 5 (Mars)  | 1.2  | true  | 0.6 |
| 6 (Mars)  | 1.5  | true  | 1.0 |
| 10 (Io)   | 2.0  | true  | 1.5 |
| 11 (Io)   | 2.5  | true  | 2.0 |
| 12 (Io)   | 3.0  | true  | 2.5 |

Wind direction is always horizontal. Negative values = left force, positive values = right force. The sign is set per level.

---

## 11. Terrain Collision

Terrain is defined as a series of line segments (polygon). Collision is detected by checking whether any of the lander's five collision points has crossed below the terrain line.

### 11.1 Detection Method

Each frame, after position update:

```javascript
for each collisionPoint in lander.collisionPoints {
  const worldPoint = rotateAndTranslate(collisionPoint, lander.angle, lander.x, lander.y);
  const terrainY = getTerrainYAtX(worldPoint.x); // interpolate terrain height at this X
  if (worldPoint.y >= terrainY) {
    // Collision detected
    handleCollision(worldPoint, terrainY);
  }
}
```

### 11.2 Collision Resolution

On terrain collision, the physics engine determines whether this is a **landing** or a **crash** (delegated to SPEC-04). If it is a crash:

```javascript
function handleCrash() {
  lander.vx = 0;
  lander.vy = 0;
  lander.state = 'CRASHED';
  // Trigger crash animation (explosion particle burst)
  // Notify lives system (SPEC-05)
}
```

The lander is frozen in place. No further physics updates occur until the level resets.

---

## 12. Velocity Terminal & Speed Cap

To prevent physics from becoming uncontrollable at extreme speeds (e.g. after a long freefall with no thrust), a terminal velocity cap is applied:

```javascript
const MAX_SPEED = 400; // pixels per second (~67 m/s at our scale)
const speed = Math.sqrt(lander.vx ** 2 + lander.vy ** 2);
if (speed > MAX_SPEED) {
  const scale = MAX_SPEED / speed;
  lander.vx *= scale;
  lander.vy *= scale;
}
```

This cap is applied **after** all force accumulation but **before** the position update.

---

## 13. Lander State Machine

The physics engine only runs in the `FLYING` state. Other states disable physics updates.

```
States:
  FLYING    → Normal physics active. All forces applied each frame.
  LANDED    → Physics frozen. Success animation triggers (SPEC-09).
  CRASHED   → Physics frozen. Crash animation triggers. Lives system notified (SPEC-05).
  RESETTING → Brief pause before level restart. No physics.

Transitions:
  FLYING → LANDED    : landing detection success (SPEC-04)
  FLYING → CRASHED   : terrain/debris/invalid landing collision
  CRASHED → RESETTING: after crash animation completes (~1.5s)
  RESETTING → FLYING : after reset delay (~1.0s), level reinitialises
```

---

## 14. Initial Lander State (Level Start)

At the start of each level, the lander is initialised to these values:

```javascript
lander = {
  x:     level.spawnX,      // defined per level in SPEC-12
  y:     level.spawnY,      // always near the top of the canvas
  vx:    0,
  vy:    0,
  angle: 0,                 // pointing straight up
  fuel:  level.startingFuel, // defined per level in SPEC-03/SPEC-12
  state: 'FLYING',
};
```

---

## 15. Physics Constants Summary

```javascript
const PHYSICS = {
  PIXELS_PER_METRE:  6,
  MAX_DT:            0.05,    // seconds
  MAX_SPEED:         400,     // px/s
  LANDER_MASS:       1.0,
  THRUST_POWER:      18.0,    // m/s²
  ROTATION_SPEED:    120,     // degrees/s

  GRAVITY: {
    MOON:  1.62,  // m/s²
    MARS:  3.72,
    TITAN: 1.35,
    IO:    1.80,
  },

  DRAG: {
    MOON:  0.0000,
    MARS:  0.0002,
    TITAN: 0.0008,
    IO:    0.0001,
  },
};
```

---

## 16. Out of Scope for This Spec

- Fuel consumption rate — see SPEC-03
- Landing success/fail criteria — see SPEC-04
- Debris collision specifics — see SPEC-07
- UFO collision and fuel drain — see SPEC-08
- Terrain polygon definitions — see SPEC-12
- Per-level wind parameters — see SPEC-12

---

## 17. Resolved Decisions

These questions were raised during drafting and are now closed.

| # | Question | Decision | Rationale |
|---|----------|----------|-----------|
| 1 | Should thrust power vary between planets? | **No — constant across all planets** | Keeps the control model predictable. The challenge of each planet comes from gravity, wind, and hazards — not from relearning thrust feel. |
| 2 | Should the canvas scale on high-DPI screens? | **No — fixed resolution (900×600)** | Avoids DPI-dependent physics bugs. `PIXELS_PER_METRE` is a single constant and never needs to adjust at runtime. Visual sharpness is a future polish concern. |
| 3 | What happens at the left/right canvas edges? | **Horizontal wrapping** | The lander reappears seamlessly on the opposite side. Prevents the player being trapped against a wall and allows terrain designs that reward wrap-around navigation. |

---

*End of SPEC-01-Physics*
