# SPEC-11 — HUD & UI
**Game:** Lunar Lander  
**Version:** 1.0  
**Status:** Draft  
**Depends on:** GDD v1.2, SPEC-03-Fuel, SPEC-05-LivesAndGameOver, SPEC-09-SuccessAnimation, SPEC-10-ScoringAndStars  
**Used by:** SPEC-12-LevelDefinitions

---

## 1. Purpose

This spec defines every on-screen element visible to the player — the in-flight HUD, all menu screens, transitions between screens, and the layout conventions that unify them. It is the authoritative reference for what goes where on screen and how it behaves. Rendering implementation lives in the developer's hands; this spec defines the contract.

---

## 2. Canvas & Layout Foundation

```javascript
const CANVAS_WIDTH  = 900; // pixels
const CANVAS_HEIGHT = 600; // pixels
```

All coordinates in this spec are in canvas pixels at this base resolution. The canvas is rendered at its native resolution — no DPI scaling, no responsive resize in v1.0.

**Rendering layer order** (bottom to top):
1. Background (space, stars)
2. Terrain fill
3. Terrain edge line + landing pads + beacon
4. Debris
5. UFO trail
6. UFO
7. Lander
8. Thrust particles
9. Wind particles
10. Crash/success particle effects
11. HUD elements
12. Screen overlay (letterbox, fade, pause dim)
13. Menu screens (when active)

---

## 3. In-Flight HUD

The in-flight HUD is always rendered during `gameState = 'FLYING'` or `'PAUSED'`. It is hidden during `SUCCESS_ANIMATION` (replaced by the skip hint) and `CRASHED`/`RESETTING` (hidden entirely until restart).

### 3.1 Fuel Bar

| Property | Value |
|----------|-------|
| Position | Top-left: x=16, y=16 |
| Width (full) | 160px |
| Height | 14px |
| Background colour | `#222233` (dark empty state) |
| Border | 1px `#444455` |

Fill colour transitions from SPEC-03 §7.1:
- >50% fuel: `#00FF88`
- 25–50%: `#FFB800`  
- <25%: `#FF4444` (flashing at 2Hz)

Numeric readout sits immediately right of the bar:
- Position: x=184, y=16
- Format: `Math.floor(lander.fuel)` — e.g. `"742"`
- Font: 12px monospace, `#FFFFFF`

Label above bar:
- Position: x=16, y=12 (above bar top edge)
- Text: `"FUEL"`
- Font: 10px, `#888888`, uppercase

### 3.2 Velocity Indicator

| Property | Value |
|----------|-------|
| Position | Top-left: x=16, y=52 |
| Layout | Two lines: vertical speed, horizontal speed |
| Font | 12px monospace, `#FFFFFF` |

```
VERT   ↓ 12.4 m/s
HORIZ  → 3.1 m/s
```

- Vertical speed: `(lander.vy / PIXELS_PER_METRE).toFixed(1)` with a `↓` arrow
- Horizontal speed: `(Math.abs(lander.vx) / PIXELS_PER_METRE).toFixed(1)` with `→` or `←` based on sign
- Label colour: `#888888`; value colour: `#FFFFFF`
- Values go **red** (`#FF4444`) when they exceed 80% of the level's respective landing threshold — early warning the player is approaching crash-speed territory

### 3.3 Altitude Indicator

| Property | Value |
|----------|-------|
| Position | Top-left: x=16, y=82 |
| Font | 12px monospace |
| Format | `"ALT  42.3 m"` |

Value uses `getAltitudeMetres()` from SPEC-06 §7. Colour is `#FFFFFF` normally; turns `#FFB800` (amber) when altitude drops below 60m (10px per metre → 360px), giving the player a low-altitude warning.

### 3.4 Angle Indicator

| Property | Value |
|----------|-------|
| Position | Top-left: x=16, y=102 |
| Font | 12px monospace |
| Format | `"TILT  2.4°"` or `"TILT  OK"` if angle < 1° |

Value uses `Math.abs(normalisedAngle(lander.angle)).toFixed(1)`. Colour is `#FFFFFF` normally; turns `#FF4444` (red) when angle exceeds 4° — warning the player they are approaching the ±5° crash threshold.

### 3.5 Lives Display

| Property | Value |
|----------|-------|
| Position | Top-right, rightmost icon at x=884, y=16 (icons go left from here) |
| Icon size | 16×20px (lander sprite at ~40% scale) |
| Icon spacing | 6px between icons |
| Icon colour | `#E8E8E8` (same as lander) |

States per SPEC-05 §8:
- Normal: all icons at full opacity
- Life lost: rightmost icon fades out over 0.3s
- Critical (1 life): remaining icon pulses 100% → 70% opacity at 1Hz

### 3.6 Level Name Banner

| Property | Value |
|----------|-------|
| Position | Top-centre: x=450 (centred), y=24 |
| Font | 16px, `#FFFFFF`, centred |
| Text | e.g. `"LEVEL 7 — TITAN"` |
| Behaviour | Fades in on level start (0.3s), fades out after 3.0s |

```javascript
function updateLevelBanner(elapsed) {
  if (elapsed < 0.3) return elapsed / 0.3;           // fade in
  if (elapsed < 3.0) return 1.0;                     // hold
  if (elapsed < 3.5) return 1.0 - (elapsed - 3.0) / 0.5; // fade out
  return 0;                                           // hidden
}
```

### 3.7 Wind Direction Indicator

Shown only on wind levels (4–6, 10–12). Position: top-centre, below level banner when faded.

| Property | Value |
|----------|-------|
| Position | x=450, y=50 (centred) |
| Size | Arrow icon ~20px wide |
| Colour | `#88AACC` |

Displays a left or right arrow with strength text: `"← WIND MED"`. Strength labels: `LOW` (<1.0 m/s²), `MED` (1.0–2.0), `HIGH` (>2.0). The arrow direction matches the force direction.

Gust warning overlay (from SPEC-07 §3.3): `"⚡ GUST"` text appears in amber (`#FFB800`) for 400ms at x=450, y=70 when gust threshold exceeded.

### 3.8 Crash Reason Flash

On crash, a brief crash reason message appears centred on screen for 1.0s, then fades:

| Crash Reason | Display Text |
|--------------|-------------|
| `TOO_FAST_VERTICAL` | `"TOO FAST"` |
| `TOO_FAST_LATERAL`  | `"LATERAL DRIFT"` |
| `BAD_ANGLE`         | `"BAD ANGLE"` |
| `LEGS_OFF_PAD`      | `"MISSED PAD"` |
| `UPSIDE_DOWN`       | `"UPSIDE DOWN"` |
| `TERRAIN_CONTACT`   | `"CRASHED"` |
| `MULTI`             | `"TOO FAST / BAD ANGLE"` (or similar combination) |

- Font: 28px bold, `#FF4444`
- Position: canvas centre x=450, y=280
- Duration: visible 0.8s, fades over 0.4s
- Rendered above HUD layer, below overlay layer

### 3.9 Skip Hint (Success Animation)

During `SUCCESS_ANIMATION`, all HUD is hidden except:

| Property | Value |
|----------|-------|
| Text | `"Space / Esc to skip"` |
| Position | Bottom-right: x=880, y=576 (right-aligned) |
| Font | 11px, `#666666` |
| Appears | 1.0s after animation start (opacity fades in over 0.3s) |

---

## 4. Pause Menu

Activated by `Escape` or `P` during flight. Rendered over a 50% dark overlay on the game canvas. The game world is still visible behind it.

| Element | Position | Description |
|---------|----------|-------------|
| "PAUSED" heading | x=450, y=200 (centred) | 32px bold, `#FFFFFF` |
| Resume | x=450, y=270 | Menu item — highlighted on selection |
| Restart Level | x=450, y=310 | Menu item |
| Level Select | x=450, y=350 | Menu item |
| Main Menu | x=450, y=390 | Menu item |

**Navigation:** `ArrowUp`/`ArrowDown` move selection highlight. `Enter`/`Space` confirm. `Escape` closes pause (equivalent to Resume).

Menu item style:
- Normal: 18px, `#AAAAAA`
- Selected: 18px, `#FFFFFF` with `▶` prefix and subtle background highlight (`rgba(255,255,255,0.08)`, full width, 32px tall)

---

## 5. Main Menu

Shown at game start and when navigating "to main menu" from any other screen.

| Element | Position | Description |
|---------|----------|-------------|
| Game title | x=450, y=160 | `"LUNAR LANDER"` — 48px bold, `#E8E8E8`, centred |
| Tagline | x=450, y=200 | `"Land softly. Land precisely."` — 16px, `#666666` |
| Total stars | x=450, y=230 | `"★ 14 / 36"` — 14px, `#FFB800` |
| Play | x=450, y=310 | Primary CTA — 22px, highlighted |
| How to Play | x=450, y=355 | 18px |
| Credits | x=450, y=395 | 18px |

Background: animated star field (slow parallax drift). No terrain visible on main menu.

---

## 6. Level Select Screen

A 4×3 grid of level cells (4 columns, 3 rows).

**Grid layout:**
- Grid starts at: x=90, y=120
- Cell size: 180×120px
- Cell gap: 12px
- Grid width: (4×180) + (3×12) = 756px — centred in 900px canvas with 72px margins

**Each cell contains:**
- Level number: top-left, 12px, `#888888`
- Environment name: 10px, `#666666` (e.g. "MOON", "MARS")
- Star display: 3 star icons (24px total), centred bottom of cell
  - Filled star: `#FFB800`
  - Empty star: `#444444`
- Locked overlay: padlock icon centred, cell bg `rgba(0,0,0,0.6)` if locked

**Header:**
- `"SELECT LEVEL"` — 24px bold, x=450, y=70
- `"★ 14 / 36"` — 14px `#FFB800`, x=450, y=100

**Navigation:**
- Arrow keys move selection between cells
- `Enter`/`Space` launches selected (unlocked) level
- `Escape` returns to main menu

Selected cell: border colour `#FFFFFF` 2px, slight brightness boost.

---

## 7. Level Complete (Results Screen)

Shown after the success animation fade.

| Element | Position | Description |
|---------|----------|-------------|
| Level name | x=450, y=100 | `"LEVEL 3 — MOON"` — 20px, `#888888` |
| "COMPLETE" | x=450, y=140 | 36px bold, `#FFFFFF` |
| Star display | x=450, y=200 | 3 large star icons (48px each), centred |
| Fuel remaining | x=340, y=280 | `"FUEL  58%"` left-aligned block |
| Best fuel | x=560, y=280 | `"BEST  71%"` right-aligned (shown if personal best exists) |
| Landing speed | x=340, y=310 | `"SPEED  3.7 m/s"` |
| Angle | x=560, y=310 | `"ANGLE  1.4°"` |
| Next Level button | x=450, y=390 | Primary CTA — **only shown for Levels 1–11** (not after Level 12) |
| Retry button | x=320, y=450 | Secondary — always shown |
| Level Select button | x=580, y=450 | Secondary — always shown |

**Button navigation:**
- `ArrowUp` / `ArrowDown` cycle between the available buttons (Next Level → Retry → Level Select, wrapping).
- `Enter` / `Space` activates the selected button.
- The selected button is highlighted with `#FFFFFF` text and a `▶` prefix. Default selection is **Next Level** (if shown), otherwise **Retry**.
- Next Level: saves current progress, loads the next level, applies a 0.3s fade-out / 0.3s fade-in transition, and goes directly to `FLYING`.
- Retry: resets the current level from scratch (`initShip()`) with a fade transition back to `FLYING`.
- Level Select: fades to the Level Select screen.

Star improvement animation: if stars improved over previous best, new stars glow briefly (0.8s pulse) before settling. Old star count shown in grey → new in gold.

---

## 8. Game Over Screen

Per SPEC-05 §6.1.

| Element | Position | Description |
|---------|----------|-------------|
| "GAME OVER" | x=450, y=180 | 40px bold, `#FF4444` |
| Level reached | x=450, y=240 | `"Reached Level 7"` — 20px, `#AAAAAA` |
| Stars this run | x=450, y=275 | `"★ 8 collected this run"` — 16px, `#FFB800` |
| Try Again | x=320, y=370 | Primary button |
| Level Select | x=580, y=370 | Secondary button |

---

## 9. How to Play Screen

Reached from Main Menu. Single screen, static content. `Escape` returns.

| Section | Content |
|---------|---------|
| Controls | Arrow keys / WASD diagram; thrust, rotate labels |
| Landing criteria | ±5° angle, speed thresholds, land on green pad |
| Star tips | Brief fuel/speed guidance for 2 and 3 star |
| Hazards | Wind, debris, UFO icons with one-line descriptions |

Font: 14px, `#CCCCCC`. Section headers: 16px bold, `#FFFFFF`.

---

## 10. Screen Transitions

All screen transitions use a fade-to-black (no wipes or slides).

| Transition | Duration | Trigger |
|------------|----------|---------|
| Level start (fade in) | 0.5s | After level reset sequence |
| Level end (fade out) | 0.4s | Phase 6 of success animation |
| Menu → Level (fade out/in) | 0.3s + 0.3s | Play selected from level select |
| Game Over (fade out) | 0.5s | After crash animation + 1.0s pause |
| Results → Next Level | 0.3s + 0.3s | Next Level button pressed |
| Results → Level Select | 0.3s | Level Select button pressed |

Fade implementation: a full-canvas black rectangle with opacity animated 0→1 (out) or 1→0 (in).

```javascript
function renderFadeOverlay(ctx, alpha) {
  if (alpha <= 0) return;
  ctx.fillStyle = `rgba(0, 0, 0, ${alpha})`;
  ctx.fillRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);
}
```

---

## 11. Typography Reference

| Context | Size | Weight | Colour | Font |
|---------|------|--------|--------|------|
| Game title | 48px | Bold | `#E8E8E8` | Sans-serif |
| Screen headings | 32–40px | Bold | `#FFFFFF` | Sans-serif |
| Section headings | 20–24px | Bold | `#FFFFFF` | Sans-serif |
| Menu items | 18–22px | Normal/Bold | `#AAAAAA` / `#FFFFFF` | Sans-serif |
| HUD labels | 10px | Normal | `#888888` | Monospace |
| HUD values | 12px | Normal | `#FFFFFF` / `#FF4444` | Monospace |
| Body / descriptions | 14px | Normal | `#CCCCCC` | Sans-serif |
| Secondary / metadata | 12px | Normal | `#666666` | Sans-serif |

Recommended font stack: `'Orbitron', 'Share Tech Mono', monospace` for HUD values; `'Inter', 'Helvetica Neue', sans-serif` for menus and headings. Both are free Google Fonts.

---

## 12. Colour Reference

| Element | Hex | Usage |
|---------|-----|-------|
| `#FFFFFF` | White | Primary text, selected items |
| `#E8E8E8` | Off-white | Lander, game title |
| `#AAAAAA` | Mid-grey | Unselected menu items |
| `#888888` | Dark-grey | HUD labels, secondary text |
| `#666666` | Darker-grey | Metadata, locked level text |
| `#444444` | Very dark | Empty stars, borders |
| `#222233` | Near-black | HUD backgrounds, fuel bar empty |
| `#0A0E1A` | Space black | Canvas background (most levels) |
| `#00FF88` | Green | Active landing pad, fuel safe |
| `#FFB800` | Amber | Fuel warning, stars, wind indicator |
| `#FF4444` | Red | Fuel critical, crashes, decoy pads |
| `#7AB8FF` | Blue | UFO glow |

---

## 13. Resolved Decisions

| # | Question | Decision | Rationale |
|---|----------|----------|-----------|
| 1 | Should HUD be visible during success animation? | No — hidden, only skip hint remains | The animation is a cinematic moment. HUD elements during it would break the immersion. The skip hint is the one exception — players need to know they can move on. |
| 2 | Should velocity warnings use absolute or proportional thresholds? | Proportional — 80% of level threshold | Absolute speed warnings would trigger at different moments on different planets. 80% of the level's own threshold gives consistent warning timing regardless of planet. |
| 3 | Should the crash reason be shown? | Yes — brief centred flash | Actionable feedback. Players learn from "BAD ANGLE" faster than from generic "CRASHED". The 1.0s duration is long enough to read, short enough not to obstruct the crash animation. |
| 4 | Should menus use keyboard navigation only? | Yes — no mouse in v1.0 | Mouse support is out of scope (GDD §15 implies desktop-only keyboard focus). Arrow keys + Enter matches the control scheme already established for flight. |
| 5 | Should screen transitions use cuts or fades? | Always fades — no cuts, no wipes | Fades are universally readable and never disorienting. Cuts feel jarring in a calm, atmospheric game. Wipes are stylistically wrong for this aesthetic. |
| 6 | Should "Next Level" be the default selection on results screen? | Yes — default to Next Level when available | Players completing a level almost always want to continue. Making Next Level the default selection reduces friction and rewards momentum. After Level 12 the default falls back to Retry. |
| 7 | Should Next Level be shown after Level 12? | No — omit the button entirely | There is no Level 13. Showing a disabled/greyed button would raise the question of why. Omitting it cleanly signals "you have finished the game." The Retry and Level Select options remain for replay. |

---

*End of SPEC-11-HUDandUI*
