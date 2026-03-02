
# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

This is a **specification-only project** — no implementation code exists yet. All design decisions are documented in `specs/`.

## Build Order

1. **Playable prototype first:** SPEC-01 (Physics) → SPEC-02 (Controls) → SPEC-03 (Fuel) → SPEC-04 (LandingDetection) → SPEC-06 (Terrain)
2. **Layer in remaining systems:** SPEC-07 (Hazards) → SPEC-08 (UFO) → SPEC-09 (SuccessAnimation) → SPEC-10 (ScoringAndStars) → SPEC-11 (HUDandUI)
3. **Full level data last:** SPEC-12 (LevelDefinitions)

Note: SPEC-05 (LivesAndGameOver) and SPEC-11 (HUDandUI) can be wired in alongside step 2.

## Specifications

The `specs/` directory contains the authoritative design documents:

- `Lunar_Lander_GDD.md` — Main Game Design Document (start here for scope/vision)
- `SPEC-01-Physics.md` — Physics simulation (gravity, thrust, drag, collision)
- `SPEC-02-Controls.md` — Input handling and key bindings
- `SPEC-03-Fuel.md` — Fuel consumption and management
- `SPEC-04-LandingDetection.md` — Landing success/fail criteria
- `SPEC-05-LivesAndGameOver.md` — Lives system and game flow
- `SPEC-06-Terrain.md` — Terrain generation and landing pads
- `SPEC-07-Hazards.md` — Wind, debris, atmospheric drag
- `SPEC-08-UFO.md` — UFO behaviour and interaction
- `SPEC-09-SuccessAnimation.md` — Post-landing cinematic sequences
- `SPEC-10-ScoringAndStars.md` — Star rating system and localStorage persistence
- `SPEC-11-HUDandUI.md` — All UI/HUD elements and menus
- `SPEC-12-LevelDefinitions.md` — All 12 level parameters

## Intended Tech Stack

- **Language:** Vanilla JavaScript (no framework)
- **Rendering:** HTML5 Canvas (900×600px fixed, 60fps via `requestAnimationFrame`)
- **Storage:** `localStorage` for star ratings and best fuel percentages
- **Build:** No build step planned (single HTML + JS files, or minimal bundling)
- **Audio:** Out of scope for v1.0

## Architecture (from specs)

### Game Loop & State Machine

The game uses a discrete state machine with states: `MAIN_MENU`, `LEVEL_SELECT`, `FLYING`, `PAUSED`, `CRASHED`, `RESETTING`, `SUCCESS_ANIMATION`, `RESULTS_SCREEN`, `GAME_OVER`.

### Physics Constants

- `PIXELS_PER_METRE = 6`
- `MAX_DT = 0.05` s (frame cap for stability)
- `MAX_SPEED = 400` px/s (terminal velocity)
- Thrust: `18.0` m/s²; Rotation: `120°/s`
- Gravity varies by planet: Moon `1.62`, Mars `3.72`, Titan `1.35`, Io `1.80` m/s²

### Input System

Use a centralised input state object (polled each frame, not event-driven callbacks). See SPEC-02.

### Level Structure

12 levels across 4 planets (Levels 1–3 Moon, 4–6 Mars, 7–9 Titan, 10–12 Io). Difficulty ramps via landing pad width (120px → 42px), fuel budget, and hazard presence (wind from L4, debris+UFO from L7). See SPEC-12 for all level parameters.

### Collision Detection

- Lander vs terrain: polygon-based with line interpolation
- Lander vs debris/UFO: AABB

### Persistence

Only star ratings (1–3), best fuel % per level, and unlock state are saved to `localStorage`. Nothing is persisted mid-level.

## Key Design Constraints

- Fixed canvas: 900×600px — no responsive scaling or high-DPI in v1.0
- Landing requires: vertical speed ≤ threshold, horizontal speed ≤ threshold, angle ≤ ±5°, both legs on pad
- UFO collision drains 25% of current fuel (not a crash)
- Lander legspan is 52px — the narrowest pad (42px on Io) is intentionally harder than the lander width
