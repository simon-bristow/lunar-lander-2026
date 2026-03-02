# SPEC-08 — UFO
**Game:** Lunar Lander  
**Version:** 1.0  
**Status:** Draft  
**Depends on:** GDD v1.2, SPEC-01-Physics, SPEC-03-Fuel, SPEC-07-Hazards  
**Used by:** SPEC-09-SuccessAnimation, SPEC-12-LevelDefinitions

---

## 1. Purpose

This spec defines the UFO — its movement model, collision behaviour, visual rendering, audio cues, and interaction with the success animation. The UFO is not a hazard in the traditional sense; it is a **curious entity** that creates passive jeopardy through proximity and unpredictability. It does not try to harm the player. The design intention is that the player feels watched, not hunted.

---

## 2. UFO Activation

The UFO is present on Levels 7–12 only. On all other levels `level.ufoEnabled = false` and no UFO is spawned.

```javascript
function initLevel() {
  if (level.ufoEnabled) {
    spawnUFO();
  }
}
```

One UFO per level maximum. Multiple UFOs are not permitted in v1.0.

---

## 3. UFO State Object

```javascript
const ufo = {
  x:          0,         // current canvas X position (centre)
  y:          0,         // current canvas Y position (centre)
  width:      48,        // collision bounding box width
  height:     20,        // collision bounding box height
  baseRadius: 160,       // nominal orbit radius around lander (pixels)
  orbitAngle: 0,         // current angle in the orbit cycle (radians)
  orbitSpeed: 0,         // radians per second — set at spawn (see §4.2)
  driftX:     0,         // secondary sinusoidal X offset
  driftY:     0,         // secondary sinusoidal Y offset
  driftPhase: 0,         // phase offset for drift oscillation
  glowPhase:  0,         // phase for pulsing glow animation
  trail:      [],        // last N positions for light trail rendering
  active:     false,     // false during success/crash animation
};
```

---

## 4. UFO Movement

The UFO's movement is composed of two layers:

1. **Primary orbit** — a wide circular orbit centred loosely on the lander's position
2. **Secondary drift** — sinusoidal X and Y offsets that make the orbit feel organic and imprecise

This two-layer approach produces the "lags, overshoots, and wanders" behaviour described in the GDD without any complex AI.

### 4.1 Movement Update (Each Frame)

```javascript
function updateUFO(dt) {
  if (!ufo.active) return;

  // Advance orbit angle
  ufo.orbitAngle += ufo.orbitSpeed * dt;

  // Advance drift phase (slower than orbit)
  ufo.driftPhase += 0.7 * dt;

  // Primary orbit — circular path around a lagged lander position
  const targetX = lerp(ufo.orbitCentreX, lander.x, 0.02); // lag — follows lander slowly
  const targetY = lerp(ufo.orbitCentreY, lander.y, 0.02);
  ufo.orbitCentreX = targetX;
  ufo.orbitCentreY = targetY;

  const primaryX = ufo.orbitCentreX + Math.cos(ufo.orbitAngle) * ufo.baseRadius;
  const primaryY = ufo.orbitCentreY + Math.sin(ufo.orbitAngle) * ufo.baseRadius * 0.5;
  // 0.5 vertical scale makes the orbit elliptical — wider than tall

  // Secondary drift — adds organic wandering
  const driftX = Math.sin(ufo.driftPhase * 1.3) * 40;
  const driftY = Math.cos(ufo.driftPhase * 0.9) * 25;

  ufo.x = primaryX + driftX;
  ufo.y = primaryY + driftY;

  // Clamp to airspace — UFO never descends into terrain zone
  ufo.y = Math.max(UFO_MIN_Y, Math.min(UFO_MAX_Y, ufo.y));

  // Update trail
  ufo.trail.unshift({ x: ufo.x, y: ufo.y });
  if (ufo.trail.length > UFO_TRAIL_LENGTH) ufo.trail.pop();

  // Advance glow phase
  ufo.glowPhase += 1.5 * dt; // 1.5 Hz pulse
}

function lerp(a, b, t) {
  return a + (b - a) * t;
}
```

### 4.2 Orbit Speed by Level

The UFO moves faster on Io levels, reflecting increased alien curiosity:

| Levels | Environment | `orbitSpeed` (rad/s) | Full orbit duration |
|--------|-------------|---------------------|---------------------|
| 7–9    | Titan       | 0.4                 | ~15.7 seconds       |
| 10–12  | Io          | 0.6                 | ~10.5 seconds       |

### 4.3 Airspace Bounds

The UFO is clamped vertically to the mid-to-upper airspace. It never descends near the landing pad:

```javascript
const UFO_MIN_Y = 60;                        // never above 60px from canvas top
const UFO_MAX_Y = CANVAS_HEIGHT * 0.55;      // never below 55% of canvas height (~330px)
```

At 330px, the UFO stays well above the terrain (which starts at ~400–450px in most levels). The player's critical approach phase near the ground is UFO-free — the jeopardy is in the mid-air descent, not the final landing.

### 4.4 Spawn Position

The UFO spawns at a random point on its initial orbit circle, so it does not always appear from the same direction:

```javascript
function spawnUFO() {
  ufo.orbitAngle   = Math.random() * Math.PI * 2; // random starting angle
  ufo.driftPhase   = Math.random() * Math.PI * 2; // random drift phase
  ufo.orbitCentreX = lander.x;
  ufo.orbitCentreY = lander.y - 80;              // start above lander
  ufo.orbitSpeed   = level.ufoOrbitSpeed;
  ufo.x            = ufo.orbitCentreX + Math.cos(ufo.orbitAngle) * ufo.baseRadius;
  ufo.y            = ufo.orbitCentreY;
  ufo.trail        = [];
  ufo.active       = true;
}
```

---

## 5. UFO Collision Detection

Collision uses axis-aligned bounding box (AABB), matching the debris collision approach from SPEC-07:

```javascript
function checkUFOCollision() {
  if (!ufo.active) return;

  const landerLeft   = lander.x - LANDER.width / 2;
  const landerRight  = lander.x + LANDER.width / 2;
  const landerTop    = lander.y - LANDER.height / 2;
  const landerBottom = lander.y + LANDER.height / 2;

  const ufoLeft   = ufo.x - ufo.width / 2;
  const ufoRight  = ufo.x + ufo.width / 2;
  const ufoTop    = ufo.y - ufo.height / 2;
  const ufoBottom = ufo.y + ufo.height / 2;

  const overlapping =
    landerRight  > ufoLeft   &&
    landerLeft   < ufoRight  &&
    landerBottom > ufoTop    &&
    landerTop    < ufoBottom;

  if (overlapping) {
    onUFOCollision();
  }
}
```

### 5.1 Collision Cooldown

After a collision, a 1.5-second cooldown prevents the same contact event from triggering multiple fuel drains while the lander and UFO are still overlapping:

```javascript
let ufoCollisionCooldown = 0;

function checkUFOCollision() {
  if (!ufo.active || ufoCollisionCooldown > 0) return;
  // ... AABB check ...
  if (overlapping) {
    onUFOCollision();
    ufoCollisionCooldown = 1.5; // seconds
  }
}

// In update loop:
if (ufoCollisionCooldown > 0) ufoCollisionCooldown -= dt;
```

### 5.2 On Collision

```javascript
function onUFOCollision() {
  // 1. Drain fuel (SPEC-03 §5)
  applyUFOFuelDrain();

  // 2. UFO continues unaffected — no state change on the UFO
  // 3. Brief UFO visual flash (see §6.3)
  ufo.collisionFlash = true;
  setTimeout(() => { ufo.collisionFlash = false; }, 300);
}
```

The UFO does not react to the collision. It continues its orbit exactly as before — it noticed the lander, not the collision.

---

## 6. Visual Rendering

The UFO is rendered each frame in two passes: glow layer first, then the saucer body on top.

### 6.1 Saucer Shape

The UFO is drawn as two overlapping ellipses — a wide flat disc and a smaller dome:

```javascript
function renderUFO(ctx) {
  ctx.save();
  ctx.translate(ufo.x, ufo.y);

  // --- Glow layer ---
  const glowAlpha = 0.3 + Math.sin(ufo.glowPhase) * 0.15; // 0.15–0.45 pulsing
  const gradient = ctx.createRadialGradient(0, 0, 0, 0, 0, 40);
  gradient.addColorStop(0, `rgba(122, 184, 255, ${glowAlpha})`);
  gradient.addColorStop(1, 'rgba(122, 184, 255, 0)');
  ctx.fillStyle = gradient;
  ctx.beginPath();
  ctx.ellipse(0, 0, 40, 40, 0, 0, Math.PI * 2);
  ctx.fill();

  // --- Saucer disc (flat ellipse) ---
  ctx.globalAlpha = 0.85; // slightly transparent per GDD
  ctx.fillStyle = '#C8E8FF';
  ctx.beginPath();
  ctx.ellipse(0, 2, 24, 8, 0, 0, Math.PI * 2);
  ctx.fill();

  // --- Dome (smaller ellipse on top) ---
  ctx.fillStyle = '#DDEEFF';
  ctx.beginPath();
  ctx.ellipse(0, -3, 12, 8, 0, 0, Math.PI * 2);
  ctx.fill();

  // --- Underside light strip ---
  ctx.fillStyle = `rgba(122, 184, 255, ${0.5 + Math.sin(ufo.glowPhase * 2) * 0.3})`;
  ctx.beginPath();
  ctx.ellipse(0, 6, 10, 3, 0, 0, Math.PI * 2);
  ctx.fill();

  ctx.restore();
}
```

### 6.2 Light Trail

The trail renders the last 12 positions of the UFO as fading dots, helping the player anticipate movement direction:

```javascript
function renderUFOTrail(ctx) {
  ufo.trail.forEach((pos, i) => {
    const alpha = (1 - i / ufo.trail.length) * 0.4; // fades to transparent
    const radius = 3 - (i / ufo.trail.length) * 2;  // shrinks with age
    ctx.beginPath();
    ctx.arc(pos.x, pos.y, radius, 0, Math.PI * 2);
    ctx.fillStyle = `rgba(122, 184, 255, ${alpha})`;
    ctx.fill();
  });
}

const UFO_TRAIL_LENGTH = 12; // positions kept in trail
```

### 6.3 Collision Flash

On collision, the UFO briefly flashes white to acknowledge the impact (even though it is unaffected by it):

```javascript
if (ufo.collisionFlash) {
  ctx.globalAlpha = 0.6;
  ctx.fillStyle = '#FFFFFF';
  ctx.beginPath();
  ctx.ellipse(ufo.x, ufo.y + 2, 26, 9, 0, 0, Math.PI * 2);
  ctx.fill();
  ctx.globalAlpha = 1.0;
}
```

---

## 7. UFO During Success Animation

When a landing is confirmed (lander state → `LANDED`), the UFO's behaviour changes for the Titan success animation only:

```javascript
function onLandingConfirmed() {
  if (level.environment === 'TITAN') {
    ufo.active = true;          // UFO stays active during Titan animation
    ufo.orbitSpeed *= 0.3;      // slows dramatically — UFO approaches to observe
    ufo.baseRadius = 80;        // tightens orbit — creeping closer with curiosity
  } else {
    ufo.active = false;         // UFO disappears on other planets
  }
}
```

This implements the GDD §9 Titan animation note: *"If the UFO is present, it hovers curiously closer to observe the planted equipment — then drifts away."* After the 2-second hold, the UFO's `baseRadius` is gradually increased back to 160 to simulate it drifting away before the fade-to-results.

On all other UFO levels (Io, 10–12), the UFO deactivates immediately on landing — it has no role in those success animations.

---

## 8. UFO Reset

The UFO is fully reset on every level reinitialisation (crash → retry, or level advance):

```javascript
function resetUFO() {
  ufo.active           = false;
  ufo.trail            = [];
  ufo.collisionFlash   = false;
  ufoCollisionCooldown = 0;
  if (level.ufoEnabled) spawnUFO();
}
```

The UFO gets a new random spawn angle on every reset — it never starts in the same position twice.

---

## 9. UFO Constants Summary

```javascript
const UFO_CONSTANTS = {
  WIDTH:          48,     // collision bbox width (px)
  HEIGHT:         20,     // collision bbox height (px)
  BASE_RADIUS:    160,    // nominal orbit radius (px)
  ORBIT_ELLIPSE:  0.5,    // vertical scale of orbit (makes it elliptical)
  DRIFT_AMPLITUDE_X: 40,  // max secondary X drift (px)
  DRIFT_AMPLITUDE_Y: 25,  // max secondary Y drift (px)
  ORBIT_SPEED: {
    TITAN: 0.4,           // rad/s
    IO:    0.6,           // rad/s
  },
  MIN_Y:          60,     // px from canvas top
  MAX_Y:          330,    // px from canvas top (~55% canvas height)
  TRAIL_LENGTH:   12,     // positions kept
  GLOW_SPEED:     1.5,    // Hz
  COLLISION_COOLDOWN: 1.5, // seconds
  COLLISION_FLASH_MS: 300, // ms
  COLOURS: {
    BODY:  '#C8E8FF',
    DOME:  '#DDEEFF',
    GLOW:  '#7AB8FF',
  },
};
```

---

## 10. Resolved Decisions

| # | Question | Decision | Rationale |
|---|----------|----------|-----------|
| 1 | Should the UFO track the lander precisely or loosely? | Loosely — lagged orbit centre with secondary drift | Precise tracking would feel aggressive and unfair. A lagged orbit that overshoots and wanders feels genuinely curious — it's interested in the lander but not hunting it. |
| 2 | Should UFO collision cause a crash or fuel drain? | Fuel drain (18.75% of current fuel) | A crash would be instant death from a non-hostile entity — too punishing and frustrating. Fuel drain is punishing but survivable, and the diminishing-returns nature means even repeated collisions don't guarantee a run-ending outcome. The original 25% was reduced by 25% (to 18.75%) after playtesting found it too severe. |
| 3 | Should there be a collision cooldown? | Yes — 1.5 seconds | Without a cooldown, a single collision event while the two objects are overlapping would drain fuel every frame. A 1.5s cooldown ensures each distinct contact is one event. |
| 4 | Should the UFO be visible during success animations? | Titan only — and it behaves differently | On Titan, the UFO drifting closer to inspect the planted equipment is narratively right and adds a memorable moment. On Io, the astronaut needs to get back to the ship quickly — the UFO's presence would undercut the urgency. |
| 5 | Should the UFO react to collisions? | No — continues orbiting unaffected | The UFO's indifference is part of its character. It's not hurt or startled. It simply continues being curious. This makes it feel more alien than a responsive enemy would. |

---

*End of SPEC-08-UFO*
