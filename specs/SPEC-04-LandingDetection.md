# SPEC-04 — Landing Detection
**Game:** Lunar Lander  
**Version:** 1.0  
**Status:** Draft  
**Depends on:** GDD v1.2, SPEC-01-Physics, SPEC-02-Controls, SPEC-03-Fuel  
**Used by:** SPEC-05-LivesAndGameOver, SPEC-09-SuccessAnimation, SPEC-10-ScoringAndStars, SPEC-12-LevelDefinitions

---

## 1. Purpose

This spec defines the complete logic for determining whether a contact event between the lander and the terrain is a **successful landing** or a **crash**. It covers the four landing criteria, the evaluation sequence, per-level thresholds, the crash path, and the data snapshot passed to the scoring system on success. This is the most consequential decision point in the game — getting it wrong either frustrates players with false crashes or cheapens success with false landings.

---

## 2. When Landing Detection Runs

Landing detection is triggered by the terrain collision check in the physics game loop (SPEC-01 §11, step 11). It does not run on every frame — only when a collision point has crossed the terrain surface.

```javascript
// Physics loop step 11 — called only when collision is detected
function handleTerrainContact(contactPoint, terrainY) {
  const onPad = isContactOnLandingPad(contactPoint.x);
  if (onPad) {
    evaluateLanding(); // may result in LANDED or CRASHED
  } else {
    triggerCrash('TERRAIN_CONTACT'); // not on pad = always crash
  }
}
```

The contact point used is whichever of the five collision polygon points first penetrated the terrain. In a normal descent (lander upright), this will always be one of the two leg tips.

---

## 3. Landing Pad Contact Check

Before evaluating landing criteria, the system checks whether the contact point is within a landing pad's horizontal bounds.

```javascript
function isContactOnLandingPad(contactX) {
  for (const pad of level.landingPads) {
    if (contactX >= pad.x && contactX <= pad.x + pad.width) {
      return true;
    }
  }
  return false;
}
```

If the contact point is **not** on a landing pad, `triggerCrash()` is called immediately — no further evaluation occurs. The four landing criteria below are only checked when contact is confirmed on a pad.

---

## 4. The Four Landing Criteria

A successful landing requires **all four** criteria to pass simultaneously at the moment of contact. If any single criterion fails, the result is a crash.

### 4.1 Criterion 1 — Vertical Velocity (Descent Speed)

The lander must be descending slowly enough not to damage the landing gear.

```javascript
const verticalOK = lander.vy <= level.maxDescentSpeed;
```

`lander.vy` is positive when moving downward (Y axis is positive down per SPEC-01 §2). `level.maxDescentSpeed` is defined per level in SPEC-12.

| Environment | Max Descent Speed (px/s) | Approx. m/s |
|-------------|--------------------------|-------------|
| Moon        | 60                        | 10.0        |
| Mars        | 54                        | 9.0         |
| Titan       | 48                        | 8.0         |
| Io          | 42                        | 7.0         |

Within each environment, the threshold tightens slightly level by level (e.g. Moon L1: 60, L2: 57, L3: 54).

### 4.2 Criterion 2 — Horizontal Velocity (Lateral Drift)

The lander must not be drifting sideways at contact. Horizontal velocity is checked as an absolute value — drift in either direction counts.

```javascript
const horizontalOK = Math.abs(lander.vx) <= level.maxLateralSpeed;
```

| Environment | Max Lateral Speed (px/s) | Approx. m/s |
|-------------|--------------------------|-------------|
| Moon        | 30                        | 5.0         |
| Mars        | 24                        | 4.0         |
| Titan       | 24                        | 4.0         |
| Io          | 18                        | 3.0         |

### 4.3 Criterion 3 — Lander Angle (Uprightness)

The lander must be close to vertical at touchdown. Angle 0° is pointing straight up. The tolerance is ±5° around vertical.

```javascript
// Normalise angle to [-180, 180] for easier comparison
const normalised = ((lander.angle + 180) % 360) - 180;
const angleOK = Math.abs(normalised) <= level.maxLandingAngle;
```

`level.maxLandingAngle` is **5 degrees** for all levels. This does not tighten per level — it is a fixed physical requirement of the landing gear geometry. The 5° tolerance exists in the GDD as a named constant and must not be changed without a GDD revision.

```javascript
const MAX_LANDING_ANGLE = 5; // degrees — fixed across all levels
```

An angle of exactly 180° (lander completely upside down) is treated as an automatic crash regardless of other criteria — see §5.3.

### 4.4 Criterion 4 — Both Leg Tips on Pad

For a structurally sound landing, **both** leg tip contact points must be over the landing pad, not just one. This prevents "edge" landings where the lander is half-on, half-off the pad.

```javascript
function bothLegsOnPad() {
  const leftLeg  = transformPoint({ x: -20, y: +22 }, lander.angle, lander.x, lander.y);
  const rightLeg = transformPoint({ x: +20, y: +22 }, lander.angle, lander.x, lander.y);
  return isContactOnLandingPad(leftLeg.x) && isContactOnLandingPad(rightLeg.x);
}
```

Note: this check uses the leg tip points defined in SPEC-01 §4.1. On narrow landing pads (later levels), this criterion becomes increasingly difficult to satisfy — both legs must fit within the pad width. This is intentional level progression pressure.

---

## 5. Full Evaluation Sequence

```javascript
function evaluateLanding() {
  // Criterion 3 shortcut: upside down = instant crash
  if (isUpsideDown()) {
    triggerCrash('UPSIDE_DOWN');
    return;
  }

  const verticalOK   = lander.vy <= level.maxDescentSpeed;
  const horizontalOK = Math.abs(lander.vx) <= level.maxLateralSpeed;
  const angleOK      = Math.abs(normalisedAngle(lander.angle)) <= MAX_LANDING_ANGLE;
  const legsOnPad    = bothLegsOnPad();

  if (verticalOK && horizontalOK && angleOK && legsOnPad) {
    triggerLanding();
  } else {
    triggerCrash(getCrashReason(verticalOK, horizontalOK, angleOK, legsOnPad));
  }
}

function isUpsideDown() {
  const normalised = Math.abs(((lander.angle + 180) % 360) - 180);
  return normalised > 90; // more than 90° from upright = upside down
}
```

### 5.1 Crash Reason Codes

When a landing attempt fails, a reason code is recorded. This is used for the crash UI (SPEC-11) and for future analytics/debugging.

| Code | Condition Failed |
|------|-----------------|
| `TERRAIN_CONTACT` | Contact outside a landing pad |
| `TOO_FAST_VERTICAL` | Vertical velocity exceeded threshold |
| `TOO_FAST_LATERAL` | Horizontal velocity exceeded threshold |
| `BAD_ANGLE` | Lander angle outside ±5° |
| `LEGS_OFF_PAD` | One or both legs not fully on pad |
| `UPSIDE_DOWN` | Lander angle >90° from vertical |
| `MULTI` | Multiple criteria failed simultaneously |

```javascript
function getCrashReason(vOK, hOK, aOK, lOK) {
  const failures = [
    !vOK && 'TOO_FAST_VERTICAL',
    !hOK && 'TOO_FAST_LATERAL',
    !aOK && 'BAD_ANGLE',
    !lOK && 'LEGS_OFF_PAD',
  ].filter(Boolean);
  return failures.length > 1 ? 'MULTI' : failures[0];
}
```

---

## 6. Successful Landing Path

When all four criteria pass, `triggerLanding()` is called:

```javascript
function triggerLanding() {
  // 1. Freeze physics
  lander.vx = 0;
  lander.vy = 0;
  lander.state = 'LANDED';

  // 2. Snap lander to upright on the pad surface
  lander.angle = 0;
  lander.y = getPadSurfaceY(lander.x) - LANDER.height / 2;

  // 3. Take landing snapshot for scoring
  const snapshot = takeLandingSnapshot();

  // 4. Notify scoring system
  scoringSystem.onLanding(snapshot);

  // 5. Trigger success animation (SPEC-09)
  successAnimation.begin(snapshot);
}
```

### 6.1 Landing Snapshot

The snapshot captures all performance data at the exact moment of landing, before any state changes. This is the single authoritative record passed to the scoring system (SPEC-10).

```javascript
function takeLandingSnapshot() {
  return {
    levelId:          level.id,
    fuelPercent:      fuelPercent(),             // from SPEC-03
    verticalSpeed:    lander.vy,                 // px/s at contact
    horizontalSpeed:  Math.abs(lander.vx),       // px/s at contact
    angle:            Math.abs(normalisedAngle(lander.angle)), // degrees from vertical
    maxDescentSpeed:  level.maxDescentSpeed,     // for % calculation in scoring
    timestamp:        Date.now(),
  };
}
```

### 6.2 Lander Snap-to-Upright

On a successful landing, the lander is snapped to angle=0 and positioned precisely on the pad surface. This ensures the success animation (SPEC-09) starts from a clean, visually correct position regardless of the exact angle at touchdown (which could be anywhere within the ±5° tolerance).

---

## 7. Crash Path

When any criterion fails, `triggerCrash(reason)` is called:

```javascript
function triggerCrash(reason) {
  // 1. Freeze physics
  lander.vx = 0;
  lander.vy = 0;
  lander.state = 'CRASHED';

  // 2. Record crash reason
  lander.crashReason = reason;

  // 3. Trigger crash animation (explosion particle burst at contact point)
  crashAnimation.begin(reason);

  // 4. Notify lives system after animation completes (~1.5s)
  setTimeout(() => {
    livesSystem.onCrash(); // SPEC-05
  }, CRASH_ANIMATION_DURATION_MS);
}

const CRASH_ANIMATION_DURATION_MS = 1500;
```

The lives system is notified **after** the crash animation completes, not immediately. This gives the player a moment to process what happened before the restart sequence begins.

---

## 8. Per-Level Threshold Summary

These values are defined here as a reference and confirmed as the authoritative source in SPEC-12 (Level Definitions). SPEC-12 may refine exact values during level design but must not change the structure.

| Level | Environment | Max Descent (px/s) | Max Lateral (px/s) | Max Angle (°) |
|-------|-------------|--------------------|--------------------|----------------|
| 1     | Moon        | 60                 | 30                 | 5              |
| 2     | Moon        | 57                 | 28                 | 5              |
| 3     | Moon        | 54                 | 26                 | 5              |
| 4     | Mars        | 54                 | 24                 | 5              |
| 5     | Mars        | 51                 | 22                 | 5              |
| 6     | Mars        | 48                 | 20                 | 5              |
| 7     | Titan       | 48                 | 24                 | 5              |
| 8     | Titan       | 45                 | 22                 | 5              |
| 9     | Titan       | 42                 | 20                 | 5              |
| 10    | Io          | 42                 | 18                 | 5              |
| 11    | Io          | 39                 | 16                 | 5              |
| 12    | Io          | 36                 | 14                 | 5              |

Max angle is constant at 5° across all levels — it represents a physical constraint of the lander's landing gear, not an arbitrary difficulty setting.

---

## 9. Edge Cases

| Scenario | Handling |
|----------|----------|
| Lander contacts pad edge at exactly threshold speed | Pass — boundary is inclusive (`<=`) |
| Both legs on pad but body clips terrain | Crash — body collision detected first by terrain check |
| Lander wraps horizontally and contacts pad near wrap point | Normal evaluation — pad coordinates are in canvas space, wrap is accounted for |
| Contact at exactly 5.0° angle | Pass — boundary is inclusive (`<=`) |
| Fuel reaches 0 exactly at landing frame | Valid landing — fuel state does not affect landing evaluation |
| Lander hits pad at 0 vertical speed (hovering perfectly) | Valid landing if all other criteria pass |

---

## 10. Resolved Decisions

| # | Question | Decision | Rationale |
|---|----------|----------|-----------|
| 1 | Should the angle tolerance tighten per level? | No — fixed 5° always | Angle tolerance is a physical constraint of the gear, not a difficulty knob. Tightening it would feel arbitrary and unfair. Difficulty progression comes from speed thresholds, pad width, and hazards. |
| 2 | Should a single leg on the pad count as a landing? | No — both legs must be on pad | Single-leg landings look wrong visually and feel cheap. Requiring both legs on the pad naturally makes narrow pads harder without a separate rule. |
| 3 | Should the crash reason be shown to the player? | Yes — displayed briefly on HUD | Crash reason gives the player immediate feedback on what to fix. "Too fast" vs "Bad angle" is actionable information. Defined in SPEC-11. |
| 4 | Should the lander snap to upright on success? | Yes — snap to angle=0 | A 4° landing angle looks odd in the success animation. Snapping to upright is imperceptible to the player but ensures the animation always looks correct. |

---

*End of SPEC-04-LandingDetection*
