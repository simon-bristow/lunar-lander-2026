# SPEC-03 — Fuel
**Game:** Lunar Lander  
**Version:** 1.0  
**Status:** Draft  
**Depends on:** GDD v1.2, SPEC-01-Physics, SPEC-02-Controls  
**Used by:** SPEC-04-LandingDetection, SPEC-08-UFO, SPEC-09-ScoringAndStars, SPEC-11-HUDandUI, SPEC-12-LevelDefinitions

---

## 1. Purpose

This spec defines the fuel system — how fuel is initialised per level, consumed during thrust, displayed to the player, and read by scoring and the UFO damage system. Fuel is the primary resource the player manages and the main driver of the star rating system. This spec owns the `lander.fuel` value and defines every way it can change.

---

## 2. Fuel Model

Fuel is stored as a single floating-point number on the lander object:

```javascript
lander.fuel         // current fuel — float, range [0, level.startingFuel]
lander.maxFuel      // set to level.startingFuel at level start, never changes mid-level
```

Fuel is **unitless** — it represents a normalised quantity, not litres or kg. All comparisons and thresholds are expressed as percentages of `lander.maxFuel`. This keeps level design simple and planet-agnostic.

```javascript
// Fuel percentage helper — used throughout all fuel-related systems
function fuelPercent() {
  return lander.fuel / lander.maxFuel; // returns 0.0 to 1.0
}
```

---

## 3. Starting Fuel Per Level

Starting fuel values decrease as levels progress, increasing pressure on the player. Values are defined in SPEC-12 (Level Definitions) and assigned at level initialisation:

```javascript
lander.fuel    = level.startingFuel;
lander.maxFuel = level.startingFuel;
```

Reference starting fuel values by environment (exact values confirmed in SPEC-12):

| Environment | Levels | Starting Fuel | Design Intent |
|-------------|--------|---------------|---------------|
| Moon        | 1–3    | 1000          | Generous — learning the controls |
| Mars        | 4–6    | 800           | Tighter — wind means more corrective burns |
| Titan       | 7–9    | 700           | Scarce — UFO and drag demand precise flying |
| Io          | 10–12  | 600           | Very scarce — every burn must count |

Within each environment, fuel decreases slightly level by level (e.g. Moon: 1000 → 950 → 900). Exact per-level values in SPEC-12.

---

## 4. Fuel Consumption

Fuel is consumed every frame that the thruster is firing AND `lander.fuel > 0`.

```javascript
// Called each frame during physics update (step 3 in SPEC-01 game loop)
if (input.thrust && lander.fuel > 0) {
  lander.fuel -= FUEL_BURN_RATE * dt;
  lander.fuel = Math.max(lander.fuel, 0); // clamp — never go negative
}
```

### 4.1 Burn Rate

```javascript
const FUEL_BURN_RATE = 60; // fuel units per second
```

At 1000 starting fuel, the player has approximately **16.7 seconds of continuous burn** before fuel is exhausted. In practice, burn is intermittent — a typical successful Moon run uses 30–60% of fuel.

The burn rate is constant across all levels and planets. Difficulty is controlled by starting fuel amount, not burn rate.

### 4.2 Empty Tank Behaviour

When `lander.fuel` reaches 0:
- The thruster immediately stops firing, regardless of input
- The thrust force in the physics engine is not applied (SPEC-01 §6 guards on `fuel > 0`)
- The thruster flame visual extinguishes immediately
- The HUD fuel bar hits empty and enters the **critical flash state** (see §7.2)
- No audio, explosion, or special event — the lander simply falls under gravity from this point

```javascript
// Fuel guard — physics engine checks this before applying thrust
if (input.thrust && lander.fuel > 0) {
  // ... apply thrust (SPEC-01 §6)
}
// If fuel === 0, thrust block is silently skipped every frame
```

The player retains full rotation control even with zero fuel. They can still orient the lander — they just cannot generate thrust.

---

## 5. UFO Fuel Drain

When the lander collides with the UFO (SPEC-08), fuel is drained as a damage effect:

```javascript
// Called by SPEC-08 on UFO collision
function applyUFOFuelDrain() {
  const drain = lander.fuel * 0.25; // 25% of current fuel
  lander.fuel -= drain;
  lander.fuel = Math.max(lander.fuel, 0);
  triggerFuelDrainFlash(); // visual feedback — see §7.3
}
```

Key properties of UFO drain:
- Drains **25% of current fuel** — not 25% of max fuel
- Diminishing in absolute terms: first hit on full tank drains 250 units; second hit (on 750) drains 187.5 units
- Fuel cannot go below 0 from UFO drain
- Multiple collisions in a single run are possible — each takes 25% of whatever remains
- The drain is instantaneous — it happens in the same frame as collision detection

---

## 6. Fuel Persistence & Reset

Fuel does **not** persist between lives or retries. On every level start (including after a crash and restart):

```javascript
function resetLevel() {
  lander.fuel    = level.startingFuel;
  lander.maxFuel = level.startingFuel;
  // ... other lander properties reset
}
```

Fuel is never shared, carried over, or modified between levels. Each level is fully self-contained.

---

## 7. HUD Display

### 7.1 Fuel Bar

The fuel bar is a horizontal progress bar displayed in the top-left of the HUD (SPEC-11 defines exact position and dimensions).

```javascript
// Each frame:
const fillWidth = HUD.fuelBar.width * fuelPercent();
// Draw background (empty bar), then fill bar on top
```

Bar fill colour transitions based on fuel percentage:

| Fuel Remaining | Bar Colour | Hex |
|----------------|------------|-----|
| >50%           | Safe green | `#00FF88` |
| 25%–50%        | Warning amber | `#FFB800` |
| <25%           | Critical red | `#FF4444` |

Colour transition is **instant** (no gradient blend) — the colour snaps to the next state when the threshold is crossed. This makes the warning states visually unambiguous.

### 7.2 Critical Flash State

When fuel drops below 25%, the fuel bar enters a **flashing state** to demand player attention:

```javascript
const FUEL_FLASH_RATE = 2; // flashes per second (on/off cycles)

// Each frame:
if (fuelPercent() < 0.25) {
  const flashOn = Math.floor(Date.now() / (1000 / FUEL_FLASH_RATE / 2)) % 2 === 0;
  fuelBar.visible = flashOn; // bar alternates visible/hidden
}
```

The flash continues until landing (success or crash) or until fuel reaches 0, at which point the bar stays visible but empty.

### 7.3 UFO Drain Visual Feedback

On UFO fuel drain, a brief visual flash communicates the damage:

- The fuel bar briefly pulses **bright white** for 0.2 seconds
- A small numeric indicator shows the amount drained (e.g. "-187") in red, floating up from the fuel bar and fading over 0.8 seconds
- This is the only in-game event that shows a numeric fuel delta

```javascript
function triggerFuelDrainFlash() {
  fuelBar.flashWhite = true;
  setTimeout(() => { fuelBar.flashWhite = false; }, 200);
  spawnFuelDrainNumber(Math.round(drain)); // floating number effect
}
```

### 7.4 Numeric Fuel Display

A small numeric readout sits beside the fuel bar showing current fuel as a whole number:

```javascript
// Displayed value
Math.floor(lander.fuel) // e.g. "742"
```

This gives precise feedback for skilled players trying to manage fuel for 3-star runs.

---

## 8. Fuel & Star Rating

Fuel remaining at the moment of successful landing is the primary star rating input. The scoring system (SPEC-09) reads `fuelPercent()` at touchdown:

| Stars | Fuel Condition | Additional Conditions |
|-------|---------------|----------------------|
| ⭐    | Any           | Landed successfully |
| ⭐⭐  | >40% remaining | Descent speed <60% of threshold |
| ⭐⭐⭐ | >65% remaining | Speed <30% of threshold, angle <2° |

Fuel percentage is calculated at the **exact frame** of landing confirmation, before any reset occurs.

```javascript
// Called by SPEC-04 on successful landing
function getFuelPercentAtLanding() {
  return fuelPercent(); // snapshot taken here, passed to scoring system
}
```

---

## 9. Fuel Constants Summary

```javascript
const FUEL = {
  BURN_RATE:          60,     // units per second of continuous thrust
  UFO_DRAIN_PERCENT:  0.25,   // fraction of current fuel drained on UFO collision
  WARN_THRESHOLD:     0.25,   // fuel % below which bar turns red and flashes
  AMBER_THRESHOLD:    0.50,   // fuel % below which bar turns amber
  FLASH_RATE:         2,      // flashes per second in critical state
  DRAIN_FLASH_MS:     200,    // milliseconds for UFO drain white flash
  DRAIN_NUMBER_MS:    800,    // milliseconds for floating drain number fade
};
```

---

## 10. Resolved Decisions

| # | Question | Decision | Rationale |
|---|----------|----------|-----------|
| 1 | Should fuel be in real units (kg, litres) or abstract? | Abstract unitless float | Real units add complexity with no gameplay benefit. Percentages keep level design simple and planet-agnostic. |
| 2 | Should burn rate vary per planet? | Constant — 60 units/second | Difficulty comes from starting fuel amount, not burn rate. A constant burn rate keeps the control feel identical across planets. |
| 3 | Should colour transition be gradual or instant? | Instant snap at thresholds | Gradual blending is subtle and easy to miss. Instant colour change is unambiguous — the player knows exactly when they've crossed into danger. |
| 4 | Should fuel persist across lives on the same level? | No — full reset on every attempt | Persistent fuel would make the first life the most important and punish experimentation. Full reset keeps each attempt fair and independent. |

---

*End of SPEC-03-Fuel*
