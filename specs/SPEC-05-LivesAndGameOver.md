# SPEC-05 — Lives & Game Over
**Game:** Lunar Lander  
**Version:** 1.0  
**Status:** Draft  
**Depends on:** GDD v1.2, SPEC-01-Physics, SPEC-04-LandingDetection  
**Used by:** SPEC-11-HUDandUI, SPEC-12-LevelDefinitions

---

## 1. Purpose

This spec defines the lives system — how lives are initialised, decremented on crash, displayed, and how their exhaustion triggers the Game Over flow. It also defines the full reset sequence that runs between a crash and the next attempt, and the campaign win condition. This spec owns the `game.lives` value and all state transitions driven by it.

---

## 2. Lives Model

Lives are stored as a single integer on the game state object:

```javascript
game.lives      // current lives remaining — integer, range [0, STARTING_LIVES]
```

Lives are **session-scoped** — they persist across levels within a single run but are fully reset when the player starts a new campaign from the level select screen.

```javascript
const STARTING_LIVES = 3;
```

---

## 3. Initialisation

Lives are initialised when the player begins a new campaign run:

```javascript
function startCampaign() {
  game.lives       = STARTING_LIVES; // 3
  game.currentLevel = 1;
  game.sessionStars = 0;
  loadLevel(1);
}
```

Lives are **not** reset when moving between levels — only when a new campaign begins. A player who completes Level 3 with 2 lives starts Level 4 with 2 lives.

Lives are **not** reset on level retry after a crash. Retrying the same level costs the life that was lost on the crash — the next attempt begins with one fewer life than before.

---

## 4. Losing a Life

The lives system is notified by SPEC-04's crash path after the crash animation completes (~1.5s delay):

```javascript
// Called by SPEC-04 triggerCrash() after animation
function onCrash() {
  game.lives -= 1;

  if (game.lives <= 0) {
    triggerGameOver();
  } else {
    triggerLevelReset();
  }
}
```

The decrement happens exactly once per crash, after the animation. No other system decrements `game.lives`.

---

## 5. Level Reset Sequence

When the player has lives remaining after a crash, the level resets. The full sequence:

```
1. game.lives decremented  (immediate, on onCrash() call)
2. HUD life icon animates out  (0.3s fade)
3. Brief pause  (0.7s — player registers the loss)
4. Screen fades to black  (0.5s)
5. Level reinitialises  (lander, fuel, wind, debris all reset per SPEC-01/03/07)
6. Screen fades in  (0.5s)
7. gameState → 'FLYING'
8. Level name fades in on HUD  (per SPEC-11)
```

Total reset duration: approximately **2.0 seconds** from `onCrash()` call to player regaining control. This is long enough to feel consequential, short enough not to frustrate.

```javascript
function triggerLevelReset() {
  gameState = 'RESETTING';

  // HUD life icon fade
  setTimeout(() => hud.animateLifeLost(), 0);

  // Pause + fade out
  setTimeout(() => screenTransition.fadeOut(500), 300 + 700);

  // Reinitialise + fade in
  setTimeout(() => {
    resetLevel();             // SPEC-01 §14, SPEC-03 §6
    screenTransition.fadeIn(500);
    gameState = 'FLYING';
  }, 300 + 700 + 500 + 100); // ~1600ms total before control returns
}
```

---

## 6. Game Over Flow

When `game.lives` reaches 0, the Game Over flow triggers:

```
1. game.lives = 0
2. gameState → 'GAME_OVER'
3. Crash animation completes (already playing from SPEC-04)
4. Brief pause  (1.0s)
5. Screen fades to black  (0.5s)
6. Game Over screen displays  (see §6.1)
```

### 6.1 Game Over Screen

The Game Over screen displays over a dark overlay. It contains:

- **"GAME OVER"** heading — large, centred
- **Level reached** — "Reached Level 7" 
- **Stars collected this run** — total stars earned across completed levels
- **Two buttons:**
  - **"Try Again"** — restarts the campaign from Level 1, full lives reset
  - **"Level Select"** — returns to the level select screen (previously earned stars preserved)

```javascript
function triggerGameOver() {
  gameState = 'GAME_OVER';

  setTimeout(() => {
    screenTransition.fadeOut(500);
    setTimeout(() => {
      showGameOverScreen({
        levelReached: game.currentLevel,
        starsThisRun: game.sessionStars,
      });
    }, 500);
  }, 1000); // 1s pause after crash animation before fade
}
```

### 6.2 Game Over Actions

| Button | Action |
|--------|--------|
| Try Again | `game.lives = STARTING_LIVES`, `game.currentLevel = 1`, `loadLevel(1)` |
| Level Select | Navigate to level select screen — session progress discarded, persistent stars kept |

"Try Again" resets lives and level but does **not** reset persistent star ratings. Stars earned in this run are already saved to localStorage by SPEC-10 at the moment of each successful landing.

---

## 7. Campaign Win Condition

The campaign is won when the player completes Level 12 (the final level) with at least 1 life remaining:

```javascript
function onLevelComplete(levelId) {
  if (levelId === 12) {
    triggerCampaignWin();
  } else {
    loadNextLevel(levelId + 1);
  }
}
```

### 7.1 Campaign Win Screen

Displayed after the Level 12 success animation completes:

- **"MISSION COMPLETE"** heading
- Total stars earned across all 12 levels
- Lives remaining at completion
- **"Play Again"** and **"Level Select"** buttons

The Campaign Win screen is distinct from a standard level complete screen. It is a significant moment — the player has finished the game.

---

## 8. Lives HUD Display

Lives are displayed as small lander icons in the top-right corner of the HUD (exact position defined in SPEC-11).

- Each icon represents one life
- Icons are rendered at 50% scale of the lander sprite, upright, white/light grey
- When a life is lost, the rightmost icon fades out with a 0.3s opacity transition
- At 1 life remaining, the single remaining icon pulses gently (opacity 100% → 70%, 1Hz) to signal danger

```javascript
// Lives display state — read by HUD renderer (SPEC-11)
function getLivesDisplayState() {
  return {
    count:        game.lives,
    animatingOut: lifeJustLost,   // triggers fade animation on rightmost icon
    critical:     game.lives === 1, // triggers pulse animation
  };
}
```

---

## 9. No Mid-Level Life Replenishment

Lives cannot be gained mid-level or between levels in v1.0. The GDD notes a "future upgrade system may allow extra lives as rewards" — this is explicitly out of scope and must not be implemented in v1.0. The lives system is designed with a `game.lives += 1` path available for future use but it is never called.

---

## 10. Session vs. Persistent State

| Data | Scope | Storage |
|------|-------|---------|
| `game.lives` | Session only | In-memory, reset on new campaign |
| `game.currentLevel` | Session only | In-memory |
| `game.sessionStars` | Session only | In-memory, used for Game Over screen |
| Star ratings per level | Persistent | localStorage (SPEC-10) |
| Best fuel % per level | Persistent | localStorage (SPEC-10) |

Lives are never written to localStorage. If the player closes the browser mid-run, their lives are lost. On return, they see the level select screen with their persistent star ratings intact.

---

## 11. State Machine Transitions Owned by This Spec

This spec drives the following game state transitions (complementing SPEC-01's physics state machine):

```
CRASHED   → RESETTING   : lives > 0, after crash animation (~1.5s)
RESETTING → FLYING      : level reset complete (~2.0s from CRASHED)
CRASHED   → GAME_OVER   : lives = 0, after crash animation (~1.5s)
LANDED    → CAMPAIGN_WIN: level 12 complete, after success animation
GAME_OVER → FLYING      : player selects "Try Again"
GAME_OVER → MENU        : player selects "Level Select"
```

---

## 12. Resolved Decisions

| # | Question | Decision | Rationale |
|---|----------|----------|-----------|
| 1 | Should lives reset between levels? | No — persist across levels in a run | Persistent lives create meaningful session stakes. Resetting between levels would remove all tension from the lives system. |
| 2 | Should lives reset on level retry? | No — retry costs the life already lost | The crash already cost a life. Resetting on retry would make lives meaningless — players could retry infinitely at no cost. |
| 3 | How long should the reset sequence take? | ~2.0 seconds total | Short enough not to frustrate, long enough for the life loss to feel consequential. Tested against other arcade games with similar systems. |
| 4 | Should "Try Again" on Game Over preserve stars? | Yes — stars already saved | Stars are written to localStorage at the moment of each landing. A new run can improve on them but cannot remove them. |
| 5 | Should there be a 1-life warning? | Yes — icon pulse at 1 life | Players need to know they're one crash from Game Over. A subtle pulse is less intrusive than a screen flash but clearly communicates the danger. |

---

*End of SPEC-05-LivesAndGameOver*
