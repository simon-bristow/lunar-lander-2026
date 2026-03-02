# SPEC-09 — Success Animation
**Game:** Lunar Lander  
**Version:** 1.0  
**Status:** Draft  
**Depends on:** GDD v1.2, SPEC-04-LandingDetection, SPEC-06-Terrain, SPEC-08-UFO  
**Used by:** SPEC-05-LivesAndGameOver, SPEC-10-ScoringAndStars, SPEC-11-HUDandUI

---

## 1. Purpose

This spec defines the success animation system — the cinematic celebration that plays after every successful landing before the results screen appears. It covers the animation state machine, per-planet variations, camera behaviour, astronaut and prop rendering, the letterbox effect, skip handling, and the handoff to the results screen. This is the emotional payoff of the game — the spec prioritises clarity and feel over technical minimalism.

---

## 2. Animation Trigger

The success animation begins immediately after `triggerLanding()` in SPEC-04. By the time this spec's systems take over, the lander is already snapped upright and frozen at `angle = 0` on the pad surface.

```javascript
// Called from SPEC-04 triggerLanding()
successAnimation.begin(snapshot);

// snapshot contains:
// { levelId, fuelPercent, verticalSpeed, horizontalSpeed, angle, maxDescentSpeed, timestamp }
```

`gameState` is set to `'SUCCESS_ANIMATION'` for the duration. All flight input is suppressed (SPEC-02 §6). Only `input.skip` is accepted.

---

## 3. Animation State Machine

Each animation phase has a fixed duration. Phases run sequentially. The system tracks current phase and elapsed time within that phase.

```javascript
const animState = {
  phase:      0,      // current phase index (0–6)
  elapsed:    0,      // seconds elapsed within current phase
  skipHinted: false,  // whether the skip hint has been shown
  skipped:    false,  // whether player has pressed skip
  environment: null,  // 'MOON' | 'MARS' | 'TITAN' | 'IO'
};

function updateAnimation(dt) {
  if (animState.skipped) return;

  animState.elapsed += dt;

  // Show skip hint after 1 second
  if (!animState.skipHinted && animState.elapsed >= 1.0) {
    animState.skipHinted = true;
    hud.showSkipHint(); // "Press Space to skip"
  }

  // Advance phase when current phase duration expires
  const phaseDuration = ANIM_PHASES[animState.phase].duration;
  if (animState.elapsed >= phaseDuration) {
    animState.elapsed -= phaseDuration;
    animState.phase += 1;
    if (animState.phase >= ANIM_PHASES.length) {
      endAnimation();
    }
  }
}
```

### 3.1 Phase Definitions

These phases apply to all environments. Per-planet visual variations are overlaid on this shared structure.

| Phase | Name | Duration | Description |
|-------|------|----------|-------------|
| 0 | `SETTLE` | 0.5s | Lander settles. Leg compression animation, dust puff at contact points. Camera zoom begins. |
| 1 | `HATCH_OPEN` | 0.5s | Top hatch swings open. Astronaut's helmet just visible in hatch. |
| 2 | `EMERGE` | 1.0s | Astronaut climbs out and drops to the surface. |
| 3 | `WALK` | 1.5s–2.0s | Astronaut walks to planting spot. Duration varies by planet (see §5). |
| 4 | `PLANT` | 0.5s | Prop planted. Dust puff. Planet-appropriate particle effect. |
| 5 | `HOLD` | 2.0s | Scene holds. Astronaut stands at scene. UFO behaviour for Titan activates here. |
| 6 | `FADE` | 0.8s | Screen fades to black, then results screen appears. |

Total animation duration (no skip): approximately **6.8–7.3 seconds** depending on planet.

---

## 4. Camera Behaviour

### 4.1 Zoom Out

At Phase 0 (`SETTLE`), the camera begins a slow zoom out from gameplay zoom (1.0×) to cinematic zoom (0.75×). The zoom reveals more of the surrounding terrain and sky, widening the scene.

```javascript
const CINEMATIC_ZOOM = 0.75;
const ZOOM_DURATION  = 1.5; // seconds to reach cinematic zoom (spans phases 0–1)

function getCameraZoom(totalElapsed) {
  const t = Math.min(totalElapsed / ZOOM_DURATION, 1.0);
  return 1.0 - (1.0 - CINEMATIC_ZOOM) * easeOut(t); // smooth deceleration
}

function easeOut(t) {
  return 1 - Math.pow(1 - t, 2); // quadratic ease-out
}
```

The camera pivot point is the lander's landing position — zoom out is centred on the lander, not the canvas centre. This ensures the lander stays visible throughout.

### 4.2 Letterbox

At the start of Phase 0, black bars animate in from the top and bottom edges, creating a 2.35:1 cinematic aspect ratio feel. This reinforces the "cutscene" context.

```javascript
const LETTERBOX_HEIGHT = 48;   // pixels per bar (top and bottom)
const LETTERBOX_IN_DURATION = 0.4; // seconds to fully extend bars

function getLetterboxProgress(elapsed) {
  return Math.min(elapsed / LETTERBOX_IN_DURATION, 1.0);
}
// Render two filled black rectangles at canvas top and bottom
// scaled by getLetterboxProgress()
```

Letterbox bars remain until the fade-to-black of Phase 6.

---

## 5. Astronaut

The astronaut is a simple silhouette figure — flat geometric shapes matching the lander's clean modern style. No detailed limb articulation. Movement is expressed through position and posture keyframes interpolated over time.

### 5.1 Astronaut Spawn Position

The astronaut emerges from the lander's hatch at the top of the lander body (world position: lander centre, Y offset −22px). On Phase 2 (`EMERGE`), the astronaut sprite transitions from hatch position to ground level beside the lander.

```javascript
const HATCH_POSITION = { x: lander.x, y: lander.y - 22 };
const GROUND_POSITION = { x: lander.x + 18, y: pad.y - ASTRONAUT_HEIGHT };
// Astronaut exits to the right side of the lander
```

### 5.2 Walk Target

The astronaut walks from ground position to a planting spot. The planting spot is offset from the lander:

```javascript
const PLANT_OFFSET = 50; // pixels from lander centre
const plantTarget = { x: lander.x + PLANT_OFFSET, y: pad.y - ASTRONAUT_HEIGHT };
```

The planting always occurs to the right of the lander. This is a fixed convention — it keeps the scene composition predictable and ensures the lander, astronaut, and prop are always in a left-to-right reading order.

### 5.3 Per-Planet Gait

The astronaut's walk cycle is expressed as a vertical bob function applied to the astronaut's Y position during Phase 3 (`WALK`):

```javascript
// Moon — bouncy low-gravity float
function moonGait(t) {
  return -Math.abs(Math.sin(t * Math.PI * 3)) * 8; // floaty bounce, lingers at top
}

// Mars — heavy deliberate steps
function marsGait(t) {
  return -Math.abs(Math.sin(t * Math.PI * 4)) * 3; // short, firm, weighted
}

// Titan — slow deliberate movement
function titanGait(t) {
  return -Math.abs(Math.sin(t * Math.PI * 2.5)) * 4; // slow, deliberate
}

// Io — quick urgent scurry
function ioGait(t) {
  return -Math.abs(Math.sin(t * Math.PI * 6)) * 3; // rapid small steps
}
// t = 0→1 over walk phase duration
```

| Planet | Steps per Walk Phase | Vertical Bob | Feel |
|--------|---------------------|--------------|------|
| Moon   | 3                   | 8px float    | Dreamy, historic |
| Mars   | 4                   | 3px firm     | Heavy, industrial |
| Titan  | 2.5                 | 4px slow     | Deliberate, eerie |
| Io     | 6                   | 3px rapid    | Urgent, dangerous |

### 5.4 Walk Duration by Planet

| Planet | Phase 3 Duration | Rationale |
|--------|-----------------|-----------|
| Moon   | 2.0s            | Slow bounce gait takes longer |
| Mars   | 1.8s            | Moderate pace |
| Titan  | 2.0s            | Deliberately slow |
| Io     | 1.5s            | Urgent — shortest walk |

---

## 6. Props

Each planet has a unique prop planted by the astronaut during Phase 4 (`PLANT`).

| Planet | Prop | Description | Colour |
|--------|------|-------------|--------|
| Moon   | Mission flag | Pole + rectangular flag; flag unfurls from left to right over 0.3s | White/blue, `#E8E8FF` |
| Mars   | Mission beacon | Vertical spike, blinking red light at top — blinks every 0.8s after plant | Dark grey + `#FF4444` blink |
| Titan  | Sensor array | Tripod base, three antennae extending outward, small status light | `#D4A843` gold + `#00FF88` light |
| Io     | Monolith | Sleek black rectangular slab, slightly taller than astronaut | `#1A1A1A` with `#444444` edge sheen |

### 6.1 Plant Particle Effect

At the moment of planting (Phase 4 start), a brief particle burst fires at the plant location:

| Planet | Particle Colour | Count | Spread | Duration |
|--------|----------------|-------|--------|----------|
| Moon   | `#AAAAAA` grey | 8     | 30px radius | 0.4s |
| Mars   | `#D4622A` rust | 15    | 50px, drifts right (wind) | 0.6s |
| Titan  | `#D4A843` amber | 10   | 40px, rises slowly (atmosphere) | 0.7s |
| Io     | `#E8C840` yellow | 12  | 35px + ground ripple | 0.5s |

### 6.2 Ground Tremor (Io Only)

At Phase 4 on Io, a ripple effect radiates from the plant point along the terrain surface — a sine wave distortion of the terrain edge line, starting at 4px amplitude and decaying to 0 over 0.5s:

```javascript
// Applied to terrain edge rendering on Io during Phase 4
function getTerrainRipple(x, elapsed) {
  const dist = Math.abs(x - plantTarget.x);
  const decay = Math.max(0, 1 - elapsed / 0.5);
  return Math.sin(dist * 0.3 - elapsed * 20) * 4 * decay;
}
```

---

## 7. Per-Planet Additional Behaviours

### 7.1 Moon — Astronaut Salute

During Phase 5 (`HOLD`), the astronaut raises one arm in a slow salute — a static pose held for the 2-second scene hold. No animation, just a posture change between Phase 4 end and Phase 5 start.

### 7.2 Mars — Visor Shield

During Phase 4 (`PLANT`), the astronaut raises one arm to shield their visor as the red dust cloud billows. The arm returns to rest at Phase 5.

### 7.3 Titan — UFO Approach

As defined in SPEC-08 §7, the UFO slows and tightens its orbit during Phase 5. The UFO becomes visually closer and more prominent during the hold. After 1.5 seconds of the hold, the UFO's orbit radius begins expanding back to 160px — it drifts away, observed and curious, as the scene fades.

### 7.4 Io — Astronaut Jog Return

Io has no Phase 5 hold in the traditional sense. After Phase 4 (`PLANT`), the astronaut immediately turns and jogs back to the lander during Phase 5. The Phase 5 duration is still 2.0s, but the astronaut is moving for the first 1.0s and then enters the lander (descends back through the hatch) for the remaining 1.0s. The scene "hold" is of the closed lander with the planted monolith visible.

```javascript
// Io Phase 5 — astronaut jogs back
if (environment === 'IO' && animState.phase === 5) {
  if (animState.elapsed < 1.0) {
    // Jog back to lander — reverse of walk, faster pace
    const t = animState.elapsed / 1.0;
    astronaut.x = lerp(plantTarget.x, GROUND_POSITION.x, easeIn(t));
  } else {
    // Astronaut descends into hatch — hide sprite
    astronaut.visible = false;
  }
}
```

---

## 8. Skip Handling

At any point after animation start, `input.skip = true` (from Space or Escape, per SPEC-02 §8) jumps directly to the results screen:

```javascript
// Checked each frame
if (input.skip) {
  animState.skipped = true;
  endAnimation(); // immediate transition
}
```

`endAnimation()` triggers the fade-to-results regardless of which phase the animation is in. The skip is instantaneous — no partial fade or wind-down.

The skip hint ("Press Space to skip") appears after 1 second of animation and persists until the fade begins. It is a small, low-contrast text in the bottom-right corner — visible but not intrusive. Second and subsequent plays the player has already seen the animation and the hint reminds them they can move on.

---

## 9. Results Handoff

At the end of Phase 6 (`FADE`), the animation system hands off to the results screen:

```javascript
function endAnimation() {
  screenTransition.fadeOut(400); // 0.4s fade to black
  setTimeout(() => {
    gameState = 'RESULTS';
    showResultsScreen(snapshot); // snapshot from SPEC-04 landing detection
    ufo.active = false;          // ensure UFO is off on all planets
    letterbox.visible = false;
  }, 400);
}
```

The snapshot passed to `showResultsScreen()` is the same object received from SPEC-04 at animation start — it is not modified during the animation.

---

## 10. Animation Constants Summary

```javascript
const SUCCESS_ANIM = {
  PHASES: [
    { name: 'SETTLE',    duration: 0.5 },
    { name: 'HATCH_OPEN', duration: 0.5 },
    { name: 'EMERGE',    duration: 1.0 },
    { name: 'WALK',      duration: null }, // set per-planet (1.5–2.0s)
    { name: 'PLANT',     duration: 0.5 },
    { name: 'HOLD',      duration: 2.0 },
    { name: 'FADE',      duration: 0.8 },
  ],
  WALK_DURATION: { MOON: 2.0, MARS: 1.8, TITAN: 2.0, IO: 1.5 },
  CINEMATIC_ZOOM:      0.75,
  ZOOM_DURATION:       1.5,   // seconds
  LETTERBOX_HEIGHT:    48,    // pixels each bar
  LETTERBOX_IN_DUR:    0.4,   // seconds to extend
  PLANT_OFFSET:        50,    // px right of lander centre
  ASTRONAUT_HEIGHT:    30,    // px
  SKIP_HINT_DELAY:     1.0,   // seconds before hint appears
  FADE_DURATION:       0.4,   // seconds fade to black at end
  TRAIL_FADE_DUR:      0.8,   // fade duration for results fade-in
};
```

---

## 11. Resolved Decisions

| # | Question | Decision | Rationale |
|---|----------|----------|-----------|
| 1 | Should the animation be skippable from frame 1? | Yes — skip works immediately, hint appears at 1s | The hint is cosmetic only. Experienced players shouldn't be forced to wait for it. First-time players benefit from seeing the full animation — the 1s delay before the hint subtly encourages that. |
| 2 | Should astronaut use full skeletal animation or keyframed postures? | Simple posture keyframes + gait bob function | Full skeletal animation is out of scope and unnecessary for the flat geometric art style. A bob function and two or three posture states (walking, planting, holding) produce a readable and charming result. |
| 3 | Should the planting always happen to the right of the lander? | Yes — fixed convention | Consistent left-to-right composition (lander → astronaut → prop) is readable and cinematic. Variable direction would require logic to avoid terrain collisions and could produce awkward compositions. |
| 4 | Should Io have a traditional scene hold? | No — astronaut jogs back instead | The GDD tone for Io is "dangerous, urgent, triumphant." A 2-second stand-and-admire hold would undercut that. The jog-back uses the same phase slot but produces a very different emotional beat. |
| 5 | Should the UFO appear in all success animations on UFO levels? | Titan only | Titan's UFO hover-and-observe is a narratively perfect moment. Io's urgent astronaut departure is undermined by an alien audience. Different emotional registers require different choices. |

---

*End of SPEC-09-SuccessAnimation*
