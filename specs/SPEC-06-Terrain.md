# SPEC-06 — Terrain
**Game:** Lunar Lander  
**Version:** 1.0  
**Status:** Draft  
**Depends on:** GDD v1.2, SPEC-01-Physics, SPEC-04-LandingDetection  
**Used by:** SPEC-07-Hazards, SPEC-11-HUDandUI, SPEC-12-LevelDefinitions

---

## 1. Purpose

This spec defines the terrain system — how terrain is represented in memory, rendered to the canvas, queried by the collision system, and how landing pads are embedded within it. It also defines terrain visual style per planet environment and the rules for terrain authoring in SPEC-12. The terrain is the primary spatial structure of every level.

---

## 2. Terrain Data Model

Terrain is represented as an ordered array of **vertices** — (x, y) coordinate pairs that form a continuous polyline across the canvas from left edge to right edge. The area below this polyline is solid ground; above it is traversable airspace.

```javascript
// Terrain definition — stored per level in SPEC-12
level.terrain = [
  { x: 0,   y: 520 },
  { x: 120, y: 500 },
  { x: 240, y: 460 },
  { x: 360, y: 460 },   // ← flat segment — landing pad zone
  { x: 480, y: 490 },
  // ... continues to canvas right edge
  { x: 900, y: 520 },
];
```

### 2.1 Constraints

| Constraint | Rule |
|------------|------|
| First vertex | Must be at `x = 0` |
| Last vertex | Must be at `x = CANVAS_WIDTH` (900) |
| X ordering | Vertices must be strictly left-to-right — no backtracking |
| Y range | All Y values must be in `[TERRAIN_MIN_Y, CANVAS_HEIGHT]` |
| Minimum spacing | Adjacent vertices must be at least 10px apart in X |
| Landing pad segments | Must be exactly horizontal (same Y for start and end X) |
| Canvas height | 600px — terrain must leave at least 200px of airspace above the highest point |

```javascript
const CANVAS_WIDTH    = 900;
const CANVAS_HEIGHT   = 600;
const TERRAIN_MIN_Y   = 200;  // terrain cannot be higher than 200px from top
const TERRAIN_MAX_Y   = 580;  // terrain cannot be lower than 580px (leaves 20px margin)
```

---

## 3. Terrain Query — `getTerrainYAtX(x)`

The physics engine (SPEC-01 §11) calls `getTerrainYAtX(x)` on every frame to check collision. This function must be fast — it is called up to 5 times per frame (once per collision point).

```javascript
function getTerrainYAtX(x) {
  // Handle horizontal wrap (SPEC-01 §8.1)
  x = ((x % CANVAS_WIDTH) + CANVAS_WIDTH) % CANVAS_WIDTH;

  // Binary search for the terrain segment containing x
  let lo = 0;
  let hi = level.terrain.length - 2;

  while (lo < hi) {
    const mid = Math.floor((lo + hi) / 2);
    if (level.terrain[mid + 1].x <= x) {
      lo = mid + 1;
    } else {
      hi = mid;
    }
  }

  // Linear interpolation between vertices lo and lo+1
  const v0 = level.terrain[lo];
  const v1 = level.terrain[lo + 1];
  const t  = (x - v0.x) / (v1.x - v0.x);
  return v0.y + t * (v1.y - v0.y);
}
```

Binary search gives O(log n) lookup. For typical terrain with 20–40 vertices, this is effectively instant. Linear interpolation between vertices ensures the terrain surface is smooth and continuous — no gaps or spikes at vertex boundaries.

---

## 4. Landing Pads

Landing pads are flat horizontal segments of the terrain. They are defined separately from the terrain polyline but must be consistent with it — the terrain vertices at a pad's edges must produce a flat segment at the pad's Y position.

```javascript
// Landing pad definition — stored per level in SPEC-12
level.landingPads = [
  {
    id:     'pad_a',
    x:      340,        // left edge in canvas pixels
    width:  80,         // pad width in pixels
    y:      460,        // surface Y — must match terrain at this X range
    active: true,       // true = valid landing target (green), false = decoy (red)
    beacon: true,       // true = beacon light displayed above this pad
  },
];
```

### 4.1 Pad Width Progression

Pad width decreases across levels, making later levels harder. The active pad width values:

| Environment | Levels | Active Pad Width | Notes |
|-------------|--------|-----------------|-------|
| Moon        | 1–3    | 120px → 100px → 80px | Wide and forgiving |
| Mars        | 4–6    | 80px → 70px → 60px  | Moderate challenge |
| Titan       | 7–9    | 60px → 55px → 50px  | Tight — lander legspan is 52px |
| Io          | 10–12  | 50px → 46px → 42px  | Extremely narrow — less than legspan |

Note: The lander's leg span is 52px (SPEC-01 §4). On Io levels, the pad is narrower than the leg span — the player must land with the lander centred precisely on the pad for both legs to contact it.

### 4.2 Multiple Pads & Decoys

From Level 4 onward, levels may include multiple pad-shaped flat segments. Only one is the active landing target. Others are **decoys** — visually identical except for colour. The active pad is indicated by a beacon light.

```javascript
// Decoy pad — looks like a pad but landing on it is a crash
{
  id:     'pad_decoy_1',
  x:      600,
  width:  80,
  y:      490,
  active: false,   // landing here triggers triggerCrash('TERRAIN_CONTACT')
  beacon: false,
}
```

Landing on a decoy pad triggers `triggerCrash('TERRAIN_CONTACT')` (SPEC-04 §2) because the contact point is not on an `active: true` pad.

### 4.3 Beacon

The active landing pad displays a **beacon** — a vertically animated light column above the pad centre:

- A vertical line of dashed/pulsing light, ~80px tall, above the pad centre
- Colour matches the pad: `#00FF88`
- Pulse rate: 1.5Hz (visible but not distracting)
- Beacon is visible at all altitudes — it helps the player identify the target from high up
- Beacon disappears once the lander touches down (success or crash)

---

## 5. Terrain Rendering

Terrain is rendered each frame as a filled polygon. The fill extends from the terrain polyline down to the canvas bottom.

```javascript
function renderTerrain(ctx) {
  ctx.beginPath();
  ctx.moveTo(level.terrain[0].x, level.terrain[0].y);

  for (let i = 1; i < level.terrain.length; i++) {
    ctx.lineTo(level.terrain[i].x, level.terrain[i].y);
  }

  // Close the polygon at the canvas bottom
  ctx.lineTo(CANVAS_WIDTH, CANVAS_HEIGHT);
  ctx.lineTo(0, CANVAS_HEIGHT);
  ctx.closePath();

  ctx.fillStyle   = level.terrainColour;
  ctx.fill();

  // Render terrain edge line on top (slightly lighter shade for definition)
  ctx.beginPath();
  ctx.moveTo(level.terrain[0].x, level.terrain[0].y);
  for (let i = 1; i < level.terrain.length; i++) {
    ctx.lineTo(level.terrain[i].x, level.terrain[i].y);
  }
  ctx.strokeStyle = level.terrainEdgeColour;
  ctx.lineWidth   = 2;
  ctx.stroke();
}
```

### 5.1 Per-Environment Terrain Colours

| Environment | Fill Colour | Edge Colour | Background |
|-------------|-------------|-------------|------------|
| Moon        | `#C8C8C8`  | `#E0E0E0`  | `#0A0E1A` |
| Mars        | `#B5451B`  | `#D4622A`  | `#1A0A06` |
| Titan       | `#D4A843`  | `#E8C060`  | `#0D0D1A` |
| Io          | `#E8C840`  | `#F5D855`  | `#0A0E1A` |

### 5.2 Landing Pad Rendering

Landing pads are rendered on top of the terrain as flat highlighted segments:

```javascript
function renderLandingPads(ctx) {
  for (const pad of level.landingPads) {
    // Pad surface
    ctx.fillStyle   = pad.active ? '#00FF88' : '#FF4444';
    ctx.fillRect(pad.x, pad.y - 4, pad.width, 4); // 4px thick surface strip

    // Pad end markers (small vertical lines)
    ctx.fillStyle = pad.active ? '#00FF88' : '#FF4444';
    ctx.fillRect(pad.x,               pad.y - 10, 3, 10);
    ctx.fillRect(pad.x + pad.width - 3, pad.y - 10, 3, 10);

    // Beacon (active pads only)
    if (pad.beacon && pad.active) {
      renderBeacon(ctx, pad.x + pad.width / 2, pad.y);
    }
  }
}
```

---

## 6. Terrain Authoring Rules

These rules apply when defining terrain vertices in SPEC-12. They exist to ensure the terrain is fair, playable, and collision-safe.

| Rule | Requirement |
|------|-------------|
| Gradual slopes | No vertical cliff faces — maximum slope between adjacent vertices: 2:1 (rise:run) |
| Clear airspace above pad | The 120px of airspace directly above every active pad must be completely clear — no overhanging terrain |
| Spawn zone clearance | The 200px below the lander's spawn point must be open airspace |
| Reachable pad | The active pad must be reachable from the spawn point with the level's starting fuel |
| Wrap continuity | The Y value of the first vertex (x=0) and last vertex (x=900) should be similar — no jarring vertical step at the wrap point |
| Minimum flat segment | Landing pad segments must be at least as wide as the pad `width` value, exactly flat (same Y) |

### 6.1 Terrain Complexity Progression

Terrain complexity increases across levels to match the rising difficulty:

| Level Range | Vertex Count | Complexity Description |
|-------------|-------------|------------------------|
| 1–3 (Moon)  | 8–12        | Gentle rolling hills, wide flat zones |
| 4–6 (Mars)  | 12–18       | Deeper valleys, steeper walls, canyon-like sections |
| 7–9 (Titan) | 16–22       | Jagged ridgelines, narrow passages, multiple false flats |
| 10–12 (Io)  | 20–28       | Aggressive peaks, minimal flat area, volcanic formations |

---

## 7. Altitude Measurement

The HUD altitude indicator (SPEC-11) displays the distance from the lander's lowest point to the terrain directly below. This uses `getTerrainYAtX`:

```javascript
function getAltitude() {
  const lowestPoint = lander.y + LANDER.height / 2; // bottom of lander bounding box
  const terrainBelow = getTerrainYAtX(lander.x);
  return Math.max(0, terrainBelow - lowestPoint);    // in pixels
}

// Convert to display units (metres)
function getAltitudeMetres() {
  return (getAltitude() / PIXELS_PER_METRE).toFixed(1); // e.g. "42.3"
}
```

Altitude reads 0 when the lander is on or below the terrain surface. It never displays a negative value.

---

## 8. Horizontal Wrap & Terrain

Because the lander wraps horizontally (SPEC-01 §8.1), terrain is effectively a closed loop. `getTerrainYAtX` handles wrapped X coordinates (see §3). When rendering the lander near the right edge and wrapping to the left, the terrain rendering must also wrap correctly:

```javascript
// If lander is near the right edge, also check collision against
// the left-side terrain for the wrapped lander position
if (lander.x > CANVAS_WIDTH - LANDER.width) {
  checkCollisionAtX(lander.x - CANVAS_WIDTH); // wrapped position
}
```

Terrain authoring rule: the first and last vertices should have similar Y values so the wrapped terrain looks continuous (§6 rule: "Wrap continuity").

---

## 9. Resolved Decisions

| # | Question | Decision | Rationale |
|---|----------|----------|-----------|
| 1 | Should terrain be procedurally generated or hand-authored? | Hand-authored per level | Procedural generation is out of scope (GDD §15). Hand-authored terrain allows precise control over difficulty, pad placement, and fairness — especially important for the narrow Io levels. |
| 2 | Should terrain use pixel-perfect collision or interpolated polyline? | Interpolated polyline | Pixel-perfect collision is expensive and unnecessary at our resolution. Linear interpolation between vertices gives smooth, predictable, and fast collision that looks identical to the rendered terrain. |
| 3 | How should terrain height be queried — linear scan or binary search? | Binary search O(log n) | Called up to 5× per frame per collision point. Binary search is negligible cost at any terrain complexity. Linear scan would be wasteful at 20–28 vertices. |
| 4 | Should decoy pads be visually distinguishable? | Yes — red colour, no beacon | Decoys are red (`#FF4444`), active pads are green (`#00FF88`). The visual distinction is clear. The challenge is navigating to the active pad, not identifying it. |

---

*End of SPEC-06-Terrain*
