# SPEC-10 — Scoring & Stars
**Game:** Lunar Lander  
**Version:** 1.0  
**Status:** Draft  
**Depends on:** GDD v1.2, SPEC-03-Fuel, SPEC-04-LandingDetection, SPEC-09-SuccessAnimation  
**Used by:** SPEC-05-LivesAndGameOver, SPEC-11-HUDandUI, SPEC-12-LevelDefinitions

---

## 1. Purpose

This spec defines how star ratings are calculated from the landing snapshot, saved to localStorage, surfaced on the results and level-select screens, and how persistent best scores are managed. Stars are the game's primary performance signal — the spec prioritises clarity of thresholds and determinism of calculation over complexity.

---

## 2. Inputs

Star calculation uses the **landing snapshot** produced by SPEC-04 §6.1 at the exact moment of touchdown:

```javascript
snapshot = {
  levelId:         number,  // 1–12
  fuelPercent:     number,  // 0.0–1.0
  verticalSpeed:   number,  // px/s at contact (positive = descending)
  horizontalSpeed: number,  // px/s at contact (absolute value)
  angle:           number,  // degrees from vertical (0–5, always positive here)
  maxDescentSpeed: number,  // level threshold in px/s (from SPEC-04 §4.1)
  timestamp:       number,  // Date.now()
}
```

All calculations use this snapshot only. No other live game state is read during star calculation.

---

## 3. Star Rating Calculation

Stars are calculated in descending order — check 3-star first, fall through to 2-star, then 1-star. A successful landing always awards at least 1 star.

```javascript
function calculateStars(snapshot) {
  const { fuelPercent, verticalSpeed, maxDescentSpeed, angle } = snapshot;

  // Derived values
  const speedRatio = verticalSpeed / maxDescentSpeed; // 0.0–1.0 (1.0 = exactly at threshold)

  // 3 stars: exceptional landing
  if (
    fuelPercent  > 0.65 &&
    speedRatio   < 0.30 &&
    angle        < 2.0
  ) return 3;

  // 2 stars: efficient landing
  if (
    fuelPercent  > 0.40 &&
    speedRatio   < 0.60
  ) return 2;

  // 1 star: successful landing (always awarded on touchdown)
  return 1;
}
```

### 3.1 Threshold Table

| Stars | Fuel Remaining | Vertical Speed | Angle |
|-------|---------------|----------------|-------|
| ⭐    | Any           | Any (within threshold) | Any (within ±5°) |
| ⭐⭐  | >40%          | <60% of level threshold | Not checked |
| ⭐⭐⭐ | >65%          | <30% of level threshold | <2° from vertical |

Key design notes:
- Horizontal speed is **not** a separate star criterion. It is a landing requirement (SPEC-04 §4.2) but not a scoring differentiator — if the player landed, they already met the horizontal threshold.
- Angle is only checked for 3 stars. 2-star landings are rewarded for fuel efficiency and speed control, not precision.
- Star thresholds are identical across all levels — difficulty comes from the environment, not from shifting goalposts.

### 3.2 Speed Ratio Examples

At Level 1 (Moon), `maxDescentSpeed = 60 px/s`:

| Vertical Speed at Landing | speedRatio | Stars Possible |
|--------------------------|-----------|----------------|
| 58 px/s (just under threshold) | 0.97 | 1 star only |
| 35 px/s | 0.58 | Up to 2 stars (if fuel > 40%) |
| 17 px/s | 0.28 | Up to 3 stars (if fuel > 65%, angle < 2°) |

At Level 12 (Io), `maxDescentSpeed = 36 px/s`:

| Vertical Speed at Landing | speedRatio | Stars Possible |
|--------------------------|-----------|----------------|
| 35 px/s | 0.97 | 1 star only |
| 20 px/s | 0.56 | Up to 2 stars |
| 10 px/s | 0.28 | Up to 3 stars |

The same speed ratio thresholds apply at every level. A 3-star landing on Io requires the same proportional precision as on the Moon — but the absolute speeds required are lower, making fuel management harder.

---

## 4. Results Screen Display

Immediately after the success animation (SPEC-09 §9), the results screen is shown. It displays:

```
Level 3 — Complete

  ★ ★ ☆       (stars earned this attempt)

  Fuel remaining:   58%   (best: 71%)
  Landing speed:    22 m/s
  Angle at touch:   1.4°

  [Next Level]   [Retry]   [Level Select]
```

### 4.1 Results Screen Data

| Field | Source | Display Format |
|-------|--------|---------------|
| Stars this attempt | `calculateStars(snapshot)` | Filled/empty star icons |
| Fuel remaining | `snapshot.fuelPercent * 100` | "58%" |
| Best fuel (personal) | `savedData[levelId].bestFuelPercent * 100` | "71%" — shown if better run exists |
| Landing speed | `snapshot.verticalSpeed / PIXELS_PER_METRE` | "22 m/s" |
| Angle at touch | `snapshot.angle` | "1.4°" |

Landing speed is displayed in m/s (converted from px/s using `PIXELS_PER_METRE = 6`). Angle is displayed to one decimal place.

### 4.2 Star Improvement Indicator

If the stars earned this attempt are higher than the previously saved best:

```javascript
if (newStars > savedData[levelId].stars) {
  showStarImprovement(savedData[levelId].stars, newStars);
  // e.g. "★☆☆ → ★★☆" with a brief celebration animation
}
```

The improvement indicator is a brief glow/flourish on the newly earned stars, lasting ~0.8s. It only shows when the new rating is better — matching the best is not celebrated.

### 4.3 Next Level Button

"Next Level" is shown only if:
- The player has completed the level (always true on the results screen)
- The next level exists (not shown after Level 12)
- The next level is already unlocked (it will be — completing this level unlocks it)

After Level 12, "Next Level" is replaced by "Campaign Complete!" which navigates to the campaign win screen (SPEC-05 §7.1).

---

## 5. Persistence — localStorage Schema

All persistent data is stored in a single localStorage key:

```javascript
const STORAGE_KEY = 'lunarLander_v1';

// Full schema
const savedData = {
  version: 1,                    // schema version — for future migration
  levels: {
    1:  { stars: 0, bestFuelPercent: 0, unlocked: true  },
    2:  { stars: 0, bestFuelPercent: 0, unlocked: false },
    3:  { stars: 0, bestFuelPercent: 0, unlocked: false },
    // ... through level 12
    12: { stars: 0, bestFuelPercent: 0, unlocked: false },
  },
  totalStars: 0,                 // sum of best stars across all levels (0–36)
};
```

### 5.1 Level Unlock State

Level 1 is always unlocked. Level N unlocks when Level N−1 is completed with ≥1 star (any successful landing). Unlock state is written to localStorage immediately on landing — before the success animation plays.

```javascript
function onLanding(snapshot) {
  const stars = calculateStars(snapshot);
  saveResult(snapshot.levelId, stars, snapshot.fuelPercent);
  unlockNextLevel(snapshot.levelId);  // write unlock before animation
}
```

### 5.2 Save on Landing

The save is written **before** the success animation, not after. This ensures that if the player closes the browser during the animation, the result is not lost.

```javascript
function saveResult(levelId, stars, fuelPercent) {
  const saved = loadSavedData();

  const current = saved.levels[levelId];
  const improved = stars > current.stars;
  const betterFuel = fuelPercent > current.bestFuelPercent;

  if (improved)   current.stars = stars;
  if (betterFuel) current.bestFuelPercent = fuelPercent;

  // Recalculate totalStars
  saved.totalStars = Object.values(saved.levels)
    .reduce((sum, l) => sum + l.stars, 0);

  writeSavedData(saved);
}
```

### 5.3 Read/Write Helpers

```javascript
function loadSavedData() {
  const raw = localStorage.getItem(STORAGE_KEY);
  if (!raw) return getDefaultSavedData();
  try {
    return JSON.parse(raw);
  } catch {
    return getDefaultSavedData(); // corrupt data fallback
  }
}

function writeSavedData(data) {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(data));
}

function getDefaultSavedData() {
  const levels = {};
  for (let i = 1; i <= 12; i++) {
    levels[i] = { stars: 0, bestFuelPercent: 0, unlocked: i === 1 };
  }
  return { version: 1, levels, totalStars: 0 };
}
```

### 5.4 Handling Corrupt or Missing Data

If localStorage is unavailable (private browsing, storage quota exceeded) or data is corrupt, `getDefaultSavedData()` is returned. The game is fully playable without persistence — progress simply isn't saved. No error is shown to the player.

---

## 6. Level Select Screen — Star Display

The level select screen (SPEC-11) reads from `savedData` to render:

- Each level cell shows its best star rating (0–3 filled stars)
- Locked levels show a padlock icon, greyed out
- Level 1 is always unlocked and shows no lock
- Total stars shown in header: e.g. "Total: 14 / 36"

```javascript
function getLevelSelectData() {
  const saved = loadSavedData();
  return Object.entries(saved.levels).map(([id, data]) => ({
    id:        Number(id),
    stars:     data.stars,
    unlocked:  data.unlocked,
    bestFuel:  data.bestFuelPercent,
  }));
}
```

---

## 7. Main Menu — Total Stars

The main menu shows a total star count to reward completionists:

```javascript
// Displayed on main menu
const totalStars   = savedData.totalStars; // e.g. 22
const maxStars     = 36;                   // 12 levels × 3 stars
// Displayed as: "★ 22 / 36"
```

---

## 8. Scoring Constants Summary

```javascript
const SCORING = {
  // Star thresholds
  TWO_STAR_FUEL:    0.40,  // fuel % required for 2+ stars
  THREE_STAR_FUEL:  0.65,  // fuel % required for 3 stars
  TWO_STAR_SPEED:   0.60,  // speedRatio threshold for 2+ stars
  THREE_STAR_SPEED: 0.30,  // speedRatio threshold for 3 stars
  THREE_STAR_ANGLE: 2.0,   // degrees from vertical for 3 stars

  // Display
  PIXELS_PER_METRE: 6,     // for speed display conversion

  // Persistence
  STORAGE_KEY:     'lunarLander_v1',
  SCHEMA_VERSION:  1,
  MAX_STARS:       36,     // 12 levels × 3 stars
};
```

---

## 9. Resolved Decisions

| # | Question | Decision | Rationale |
|---|----------|----------|-----------|
| 1 | Should horizontal speed factor into star rating? | No — it's a landing requirement, not a star criterion | Players who successfully land have already met the horizontal threshold. Adding it as a star criterion would make 3-star runs luck-dependent in windy levels (Io) where lateral correction is constant. |
| 2 | Should star thresholds change per level? | No — same thresholds on all levels | Consistent thresholds respect the player's learned intuition. Difficulty comes from the environment making good landings harder — not from arbitrary threshold shifts. |
| 3 | Should best fuel and best stars track independently? | Yes — both tracked separately | A player might get 3 stars with 66% fuel, then later get 2 stars with 80% fuel. The star rating improves only if stars improve — but the best fuel always updates when a better fuel result is achieved. |
| 4 | When should the save be written? | Before the animation begins | If the player closes the browser during the ~7s animation, the result is already saved. Saving after the animation would create a window where progress could be lost. |
| 5 | Should data corruption be surfaced to the player? | No — silently fall back to defaults | Surfacing a "save data corrupt" message would alarm and confuse most players. Silent fallback to defaults means the game is always playable, and most players won't notice unless they've accumulated significant progress. |

---

*End of SPEC-10-ScoringAndStars*
