# SPEC-02 — Controls
**Game:** Lunar Lander  
**Version:** 1.0  
**Status:** Draft  
**Depends on:** GDD v1.2, SPEC-01-Physics  
**Used by:** SPEC-03-Fuel, SPEC-05-LivesAndGameOver, SPEC-09-SuccessAnimation, SPEC-11-HUDandUI

---

## 1. Purpose

This spec defines the complete input handling system for Lunar Lander. It covers key bindings, the input state object, how raw browser events are translated into game actions, input buffering, pause behaviour, and menu navigation. All game systems that need to know what the player is doing read from the input state object defined here — they never listen to raw browser events directly.

---

## 2. Design Principles

- **Single source of truth.** One `input` object holds all current input state. Physics, UI, and all other systems read from this object only.
- **No direct event handling in game logic.** `keydown` and `keyup` events only write to the input object. They never call physics or game logic functions directly.
- **Held-key model for flight.** Rotation and thrust respond to held keys, not keypress events. The physics loop reads input state on every frame.
- **Pressed-once model for menus.** Menu navigation and pause use single keypress detection to prevent repeated firing.

---

## 3. Input State Object

A single global `input` object is maintained for the lifetime of the game. It is updated by event listeners and read by the game loop.

```javascript
const input = {
  // Flight controls (held-key — true while key is down)
  thrust:      false,   // Main engine firing
  rotateLeft:  false,   // Rotate counter-clockwise
  rotateRight: false,   // Rotate clockwise

  // Single-press actions (true for exactly one frame, then reset to false)
  pause:       false,   // Toggle pause
  skip:        false,   // Skip success animation
  confirm:     false,   // Confirm menu selection (Enter)
  back:        false,   // Back / cancel (Escape in menus)

  // Menu navigation (single-press)
  menuUp:      false,
  menuDown:    false,
  menuLeft:    false,
  menuRight:   false,
};
```

Single-press actions are reset to `false` at the **end** of each game loop frame, after all systems have had a chance to read them. This guarantees every system sees a press exactly once.

```javascript
// Called at the END of each game loop frame
function resetSinglePressInputs() {
  input.pause      = false;
  input.skip       = false;
  input.confirm    = false;
  input.back       = false;
  input.menuUp     = false;
  input.menuDown   = false;
  input.menuLeft   = false;
  input.menuRight  = false;
}
```

---

## 4. Key Bindings

### 4.1 Flight Controls

| Action | Primary Key | Alternative Key | Input Type |
|--------|------------|-----------------|------------|
| Thrust (main engine) | `ArrowUp` | `W`, `Space` | Held |
| Rotate left (CCW) | `ArrowLeft` | `A` | Held |
| Rotate right (CW) | `ArrowRight` | `D` | Held |

All three flight controls support two key bindings simultaneously. Both bindings set the same input flag — if either key is held, the action is active.

### 4.2 System Controls

| Action | Key(s) | Input Type | Active In |
|--------|--------|------------|-----------|
| Pause / unpause | `Escape`, `P` | Single-press | In-flight only |
| Skip animation | `Space`, `Escape` | Single-press | Success animation only |
| Confirm selection | `Enter`, `Space` | Single-press | Menus only |
| Back / cancel | `Escape` | Single-press | Menus only |
| Menu up | `ArrowUp`, `W` | Single-press | Menus only |
| Menu down | `ArrowDown`, `S` | Single-press | Menus only |
| Menu left | `ArrowLeft`, `A` | Single-press | Menus only |
| Menu right | `ArrowRight`, `D` | Single-press | Menus only |

### 4.3 Key Conflict Rules

`Space` serves double duty as thrust (in-flight) and confirm (in menus). This is resolved by context — the game state determines which action `Space` triggers:

```javascript
// Space is context-sensitive
if (gameState === 'FLYING') {
  input.thrust = true;           // Space = thrust
} else if (gameState === 'MENU') {
  input.confirm = true;          // Space = confirm
} else if (gameState === 'SUCCESS_ANIMATION') {
  input.skip = true;             // Space = skip
}
```

`Escape` also serves double duty — pause in-flight, back in menus — resolved the same way.

---

## 5. Event Listener Implementation

Event listeners are attached to `window` once at game initialisation and never removed during gameplay.

```javascript
window.addEventListener('keydown', (e) => {
  // Prevent default browser behaviour for game keys
  if (GAME_KEYS.includes(e.code)) e.preventDefault();

  // Ignore key repeat events (held key fires repeatedly in browser)
  // Held-key actions are managed by our own state, not browser repeat
  if (e.repeat) return;

  handleKeyDown(e.code);
});

window.addEventListener('keyup', (e) => {
  handleKeyUp(e.code);
});
```

### 5.1 Preventing Browser Default Behaviour

Arrow keys scroll the page by default. Space bar scrolls or activates focused elements. These must be suppressed during gameplay.

```javascript
const GAME_KEYS = [
  'ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight',
  'Space', 'KeyW', 'KeyA', 'KeyS', 'KeyD',
  'Escape', 'KeyP', 'Enter',
];
```

`e.preventDefault()` is called for all game keys on `keydown`. It is **not** called on `keyup` (not needed) and **not** called when the game is on a menu screen where browser behaviour (e.g. Tab focus) may be desirable — this is handled by checking `gameState` before calling `preventDefault`.

### 5.2 handleKeyDown

```javascript
function handleKeyDown(code) {
  switch (code) {
    // --- Held flight controls ---
    case 'ArrowUp':   case 'KeyW':
      if (gameState === 'FLYING') input.thrust = true;
      if (gameState === 'MENU')   input.menuUp = true;
      break;
    case 'Space':
      if (gameState === 'FLYING')            input.thrust = true;
      if (gameState === 'MENU')              input.confirm = true;
      if (gameState === 'SUCCESS_ANIMATION') input.skip = true;
      break;
    case 'ArrowLeft':  case 'KeyA':
      if (gameState === 'FLYING') input.rotateLeft = true;
      if (gameState === 'MENU')   input.menuLeft = true;
      break;
    case 'ArrowRight': case 'KeyD':
      if (gameState === 'FLYING') input.rotateRight = true;
      if (gameState === 'MENU')   input.menuRight = true;
      break;
    case 'ArrowDown':  case 'KeyS':
      if (gameState === 'MENU') input.menuDown = true;
      break;

    // --- Single-press actions ---
    case 'Escape':
      if (gameState === 'FLYING')            input.pause = true;
      if (gameState === 'MENU')              input.back = true;
      if (gameState === 'SUCCESS_ANIMATION') input.skip = true;
      break;
    case 'KeyP':
      if (gameState === 'FLYING' || gameState === 'PAUSED') input.pause = true;
      break;
    case 'Enter':
      if (gameState === 'MENU') input.confirm = true;
      break;
  }
}
```

### 5.3 handleKeyUp

On key up, only held-key flags are cleared. Single-press flags are never set on keyup — they are set on keydown and cleared by `resetSinglePressInputs()` at frame end.

```javascript
function handleKeyUp(code) {
  switch (code) {
    case 'ArrowUp':   case 'KeyW':   case 'Space':
      input.thrust = false;
      break;
    case 'ArrowLeft':  case 'KeyA':
      input.rotateLeft = false;
      break;
    case 'ArrowRight': case 'KeyD':
      input.rotateRight = false;
      break;
  }
}
```

---

## 6. Game State & Input Context

Input handling is context-sensitive based on `gameState`. The valid states and their accepted inputs are:

| Game State | Accepted Inputs |
|------------|----------------|
| `FLYING` | thrust, rotateLeft, rotateRight, pause |
| `PAUSED` | pause (to unpause), menuUp, menuDown, confirm, back |
| `CRASHED` | None — all input ignored during crash animation |
| `RESETTING` | None — all input ignored |
| `SUCCESS_ANIMATION` | skip |
| `MENU` | menuUp, menuDown, menuLeft, menuRight, confirm, back |

Flight inputs (`thrust`, `rotateLeft`, `rotateRight`) are **forcibly cleared** when the game leaves `FLYING` state. This prevents the thruster or rotation from being active when returning to flight after a pause.

```javascript
function onLeaveFlightState() {
  input.thrust      = false;
  input.rotateLeft  = false;
  input.rotateRight = false;
}
```

---

## 7. Pause Behaviour

Pausing is toggled by `Escape` or `P` during flight only.

```javascript
if (input.pause) {
  if (gameState === 'FLYING') {
    gameState = 'PAUSED';
    onLeaveFlightState(); // clear held flight inputs
  } else if (gameState === 'PAUSED') {
    gameState = 'FLYING';
  }
}
```

While paused:
- The game loop continues running (for menu rendering)
- Physics update is skipped
- All flight inputs are ignored
- The pause menu is rendered over the game (see SPEC-11)

---

## 8. Input During Success Animation

During the success animation sequence (SPEC-09), all flight controls are disabled. Only `skip` is accepted.

```javascript
if (gameState === 'SUCCESS_ANIMATION' && input.skip) {
  skipSuccessAnimation(); // jump straight to results screen
}
```

The "Press Space to skip" hint is shown 1 second after the animation begins. Skipping before the hint appears is still valid — the hint is cosmetic only.

---

## 9. Responsiveness Requirements

These are the minimum responsiveness targets the input system must meet:

| Metric | Target |
|--------|--------|
| Input-to-physics latency | ≤ 1 frame (≤ 16.7ms at 60fps) |
| Key repeat suppression | `e.repeat` check on all keydown handlers |
| Simultaneous keys | At minimum: thrust + rotateLeft + rotateRight all active together |
| Input persistence | Held-key state survives across frames correctly — not re-evaluated each frame from scratch |

### 9.1 Simultaneous Key Handling

The player must be able to hold thrust and rotate simultaneously — this is a core gameplay requirement. The input model supports this natively since each action is an independent boolean flag.

Tested combinations that must work:
- Thrust + RotateLeft
- Thrust + RotateRight
- RotateLeft + RotateRight (both held — net rotation = 0, cancels out in physics)

---

## 10. No Gamepad / Touch Support (v1.0)

Gamepad and touch input are explicitly out of scope for v1.0 as noted in the GDD. The input system is designed so that a `gamepadInput` or `touchInput` module could be added later to write to the same `input` object without changing any downstream systems.

---

## 11. Resolved Decisions

| # | Question | Decision | Rationale |
|---|----------|----------|-----------|
| 1 | Should `Space` act as thrust, confirm, or skip? | Context-sensitive based on `gameState` | Same key, different meaning per context. Keeps the keyboard ergonomic — Space is the "do the main thing" key in all contexts. |
| 2 | Should key repeat be used for menu navigation? | No — single press only, no repeat | Menu navigation should feel deliberate. Browser key repeat delay is variable and unreliable. |
| 3 | Should inputs be polled or event-driven? | Hybrid — events write to state, physics polls state | Events alone miss held-key state. Polling alone requires per-frame key scanning. The hybrid model is the standard approach for browser games. |

---

*End of SPEC-02-Controls*
