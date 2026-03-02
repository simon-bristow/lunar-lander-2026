# SPEC-12 — Level Definitions
**Game:** Lunar Lander  
**Version:** 1.0  
**Status:** Draft  
**Depends on:** GDD v1.2, SPEC-01 through SPEC-11 (all preceding specs)  
**Used by:** Game implementation directly — this is the authoritative data source for all 12 levels

---

## 1. Purpose

This spec is the authoritative data definition for all 12 levels. It defines the level object structure, then provides the complete parameters for every level — environment, physics constants, landing thresholds, fuel, terrain vertices, landing pads, hazard configuration, and spawn position. Everything a developer needs to instantiate any level is in this document.

---

## 2. Level Object Structure

Every level is a JavaScript object conforming to this schema. All fields are required unless marked optional.

```javascript
{
  // Identity
  id:             number,       // 1–12
  name:           string,       // display name e.g. "Sea of Tranquillity"
  environment:    string,       // 'MOON' | 'MARS' | 'TITAN' | 'IO'

  // Physics (SPEC-01)
  gravity:        number,       // m/s² — planet gravity constant
  drag:           number,       // drag coefficient (0 on Moon)
  wind:           number,       // m/s² horizontal force (0 if no wind)
  gustEnabled:    boolean,
  gustAmplitude:  number,       // m/s² additional gust force (0 if no gusts)
  gustFrequency:  number,       // Hz (0 if no gusts)

  // Fuel (SPEC-03)
  startingFuel:   number,       // fuel units at level start

  // Landing thresholds (SPEC-04)
  maxDescentSpeed:  number,     // px/s — max vertical speed for valid landing
  maxLateralSpeed:  number,     // px/s — max horizontal speed for valid landing
  // maxLandingAngle is always 5° — not stored per level

  // Spawn
  spawnX:         number,       // lander start X (canvas pixels)
  spawnY:         number,       // lander start Y (canvas pixels)

  // Terrain (SPEC-06)
  terrain:        Array<{x, y}>, // ordered vertices left to right
  terrainColour:  string,        // fill hex colour
  terrainEdgeColour: string,     // edge line hex colour
  background:     string,        // sky hex colour

  // Landing pads (SPEC-06 §4)
  landingPads:    Array<{
    id, x, width, y, active, beacon
  }>,

  // Hazards (SPEC-07)
  debrisPieces:   Array<{       // empty array if no debris
    id, shape, x, y, width, height, drift, driftMin, driftMax, rotation
  }>,

  // UFO (SPEC-08)
  ufoEnabled:       boolean,
  ufoOrbitSpeed:    number,     // rad/s (ignored if ufoEnabled=false)
}
```

---

## 3. Environment Constants (Shared Across Levels)

These values are the same for all levels within an environment. They are defined here and referenced by each level definition.

```javascript
const ENV = {
  MOON:  { gravity: 1.62, drag: 0.0000, terrainColour: '#C8C8C8', terrainEdgeColour: '#E0E0E0', background: '#0A0E1A' },
  MARS:  { gravity: 3.72, drag: 0.0002, terrainColour: '#B5451B', terrainEdgeColour: '#D4622A', background: '#1A0A06' },
  TITAN: { gravity: 1.35, drag: 0.0008, terrainColour: '#D4A843', terrainEdgeColour: '#E8C060', background: '#0D0D1A' },
  IO:    { gravity: 1.80, drag: 0.0001, terrainColour: '#E8C840', terrainEdgeColour: '#F5D855', background: '#0A0E1A' },
};
```

---

## 4. Level Definitions

### Level 1 — "Sea of Tranquillity" (Moon)

**Design intent:** Tutorial level. Wide pad, low threshold speeds, generous fuel. Teach the controls.

```javascript
{
  id: 1, name: "Sea of Tranquillity", environment: 'MOON',
  gravity: 1.62, drag: 0.0000, wind: 0, gustEnabled: false, gustAmplitude: 0, gustFrequency: 0,
  startingFuel: 1000,
  maxDescentSpeed: 60, maxLateralSpeed: 30,
  spawnX: 450, spawnY: 80,
  terrain: [
    { x: 0,   y: 510 }, { x: 100, y: 495 }, { x: 200, y: 480 },
    { x: 320, y: 480 }, // pad zone flat
    { x: 460, y: 480 }, // pad zone flat
    { x: 560, y: 500 }, { x: 700, y: 515 }, { x: 820, y: 505 }, { x: 900, y: 510 },
  ],
  terrainColour: '#C8C8C8', terrainEdgeColour: '#E0E0E0', background: '#0A0E1A',
  landingPads: [
    { id: 'pad_a', x: 330, width: 120, y: 480, active: true, beacon: true },
  ],
  debrisPieces: [],
  ufoEnabled: false, ufoOrbitSpeed: 0,
}
```

### Level 2 — "Copernicus Ridge" (Moon)

**Design intent:** Introduce moderate terrain relief. Pad slightly narrower and offset toward one side.

```javascript
{
  id: 2, name: "Copernicus Ridge", environment: 'MOON',
  gravity: 1.62, drag: 0.0000, wind: 0, gustEnabled: false, gustAmplitude: 0, gustFrequency: 0,
  startingFuel: 950,
  maxDescentSpeed: 57, maxLateralSpeed: 28,
  spawnX: 200, spawnY: 80,
  terrain: [
    { x: 0,   y: 530 }, { x: 80,  y: 490 }, { x: 160, y: 455 },
    { x: 260, y: 455 }, // pad zone
    { x: 380, y: 455 },
    { x: 450, y: 430 }, { x: 540, y: 395 }, { x: 620, y: 430 },
    { x: 720, y: 495 }, { x: 820, y: 520 }, { x: 900, y: 525 },
  ],
  terrainColour: '#C8C8C8', terrainEdgeColour: '#E0E0E0', background: '#0A0E1A',
  landingPads: [
    { id: 'pad_a', x: 270, width: 100, y: 455, active: true, beacon: true },
  ],
  debrisPieces: [],
  ufoEnabled: false, ufoOrbitSpeed: 0,
}
```

### Level 3 — "Tycho Basin" (Moon)

**Design intent:** Final Moon level. Two candidate flat zones — one is the active pad, one is terrain. Pad narrowed to 80px.

```javascript
{
  id: 3, name: "Tycho Basin", environment: 'MOON',
  gravity: 1.62, drag: 0.0000, wind: 0, gustEnabled: false, gustAmplitude: 0, gustFrequency: 0,
  startingFuel: 900,
  maxDescentSpeed: 54, maxLateralSpeed: 26,
  spawnX: 450, spawnY: 80,
  terrain: [
    { x: 0,   y: 520 }, { x: 100, y: 480 }, { x: 180, y: 445 },
    { x: 240, y: 445 }, // false flat (no pad)
    { x: 310, y: 445 },
    { x: 380, y: 420 }, { x: 470, y: 400 }, { x: 560, y: 420 },
    { x: 620, y: 445 }, // active pad zone
    { x: 720, y: 445 },
    { x: 800, y: 470 }, { x: 900, y: 510 },
  ],
  terrainColour: '#C8C8C8', terrainEdgeColour: '#E0E0E0', background: '#0A0E1A',
  landingPads: [
    { id: 'pad_a', x: 630, width: 80,  y: 445, active: true,  beacon: true  },
    { id: 'pad_b', x: 248, width: 80,  y: 445, active: false, beacon: false }, // decoy
  ],
  debrisPieces: [],
  ufoEnabled: false, ufoOrbitSpeed: 0,
}
```

---

### Level 4 — "Hellas Planitia" (Mars)

**Design intent:** Introduce wind. Constant rightward push, no gusts yet. Wider valley so player can feel wind effect clearly.

```javascript
{
  id: 4, name: "Hellas Planitia", environment: 'MARS',
  gravity: 3.72, drag: 0.0002, wind: +0.8, gustEnabled: false, gustAmplitude: 0, gustFrequency: 0,
  startingFuel: 800,
  maxDescentSpeed: 54, maxLateralSpeed: 24,
  spawnX: 200, spawnY: 80,
  terrain: [
    { x: 0,   y: 500 }, { x: 120, y: 470 }, { x: 220, y: 450 },
    { x: 340, y: 450 }, // pad zone
    { x: 460, y: 450 },
    { x: 560, y: 480 }, { x: 680, y: 510 }, { x: 800, y: 520 }, { x: 900, y: 515 },
  ],
  terrainColour: '#B5451B', terrainEdgeColour: '#D4622A', background: '#1A0A06',
  landingPads: [
    { id: 'pad_a', x: 350, width: 80, y: 450, active: true, beacon: true },
  ],
  debrisPieces: [],
  ufoEnabled: false, ufoOrbitSpeed: 0,
}
```

### Level 5 — "Valles Marineris" (Mars)

**Design intent:** Introduce gusts. Leftward wind with variable gusts. Canyon-like terrain that funnels the player toward the pad. Decoy pad added.

```javascript
{
  id: 5, name: "Valles Marineris", environment: 'MARS',
  gravity: 3.72, drag: 0.0002, wind: -1.2, gustEnabled: true, gustAmplitude: 0.6, gustFrequency: 0.4,
  startingFuel: 800,
  maxDescentSpeed: 51, maxLateralSpeed: 22,
  spawnX: 450, spawnY: 80,
  terrain: [
    { x: 0,   y: 480 }, { x: 80,  y: 440 }, { x: 160, y: 400 },
    { x: 200, y: 380 }, // canyon wall left peak
    { x: 280, y: 460 }, // drop into valley
    { x: 360, y: 460 }, // pad zone
    { x: 450, y: 460 },
    { x: 530, y: 455 }, // decoy zone
    { x: 610, y: 455 },
    { x: 670, y: 410 }, { x: 760, y: 380 }, { x: 840, y: 420 }, { x: 900, y: 470 },
  ],
  terrainColour: '#B5451B', terrainEdgeColour: '#D4622A', background: '#1A0A06',
  landingPads: [
    { id: 'pad_a', x: 368, width: 70, y: 460, active: true,  beacon: true  },
    { id: 'pad_b', x: 538, width: 70, y: 455, active: false, beacon: false },
  ],
  debrisPieces: [],
  ufoEnabled: false, ufoOrbitSpeed: 0,
}
```

### Level 6 — "Olympus Approach" (Mars)

**Design intent:** Final Mars level. Strong gusting wind. Narrow pad atop a raised plateau. Multiple false flats.

```javascript
{
  id: 6, name: "Olympus Approach", environment: 'MARS',
  gravity: 3.72, drag: 0.0002, wind: +1.5, gustEnabled: true, gustAmplitude: 1.0, gustFrequency: 0.5,
  startingFuel: 750,
  maxDescentSpeed: 48, maxLateralSpeed: 20,
  spawnX: 700, spawnY: 80,
  terrain: [
    { x: 0,   y: 540 }, { x: 100, y: 510 }, { x: 180, y: 480 },
    { x: 230, y: 480 }, // false flat
    { x: 290, y: 480 },
    { x: 340, y: 450 }, { x: 400, y: 420 },
    { x: 440, y: 390 }, // plateau
    { x: 510, y: 390 }, // active pad zone
    { x: 580, y: 390 },
    { x: 620, y: 420 }, { x: 700, y: 460 },
    { x: 750, y: 460 }, // second false flat
    { x: 820, y: 460 },
    { x: 880, y: 490 }, { x: 900, y: 510 },
  ],
  terrainColour: '#B5451B', terrainEdgeColour: '#D4622A', background: '#1A0A06',
  landingPads: [
    { id: 'pad_a', x: 450, width: 60, y: 390, active: true,  beacon: true  },
    { id: 'pad_b', x: 240, width: 60, y: 480, active: false, beacon: false },
    { id: 'pad_c', x: 760, width: 60, y: 460, active: false, beacon: false },
  ],
  debrisPieces: [],
  ufoEnabled: false, ufoOrbitSpeed: 0,
}
```

---

### Level 7 — "Kraken Mare" (Titan)

**Design intent:** Introduce debris and UFO simultaneously. Strong drag, no wind. Three pieces of debris in a loose field. Wide pad to compensate for new hazards.

```javascript
{
  id: 7, name: "Kraken Mare", environment: 'TITAN',
  gravity: 1.35, drag: 0.0008, wind: 0, gustEnabled: false, gustAmplitude: 0, gustFrequency: 0,
  startingFuel: 700,
  maxDescentSpeed: 48, maxLateralSpeed: 24,
  spawnX: 450, spawnY: 80,
  terrain: [
    { x: 0,   y: 505 }, { x: 120, y: 490 }, { x: 230, y: 475 },
    { x: 310, y: 475 }, // pad zone
    { x: 420, y: 475 },
    { x: 510, y: 490 }, { x: 630, y: 510 }, { x: 750, y: 500 }, { x: 900, y: 505 },
  ],
  terrainColour: '#D4A843', terrainEdgeColour: '#E8C060', background: '#0D0D1A',
  landingPads: [
    { id: 'pad_a', x: 318, width: 60, y: 475, active: true, beacon: true },
  ],
  debrisPieces: [
    { id: 'd1', shape: 'rock',    x: 200, y: 320, width: 28, height: 22, drift: 0,    driftMin: 0,   driftMax: 0,   rotation: 15 },
    { id: 'd2', shape: 'boulder', x: 520, y: 280, width: 44, height: 38, drift: 0,    driftMin: 0,   driftMax: 0,   rotation: 5  },
    { id: 'd3', shape: 'rock',    x: 680, y: 360, width: 26, height: 20, drift: 8,    driftMin: 620, driftMax: 740, rotation: 30 },
  ],
  ufoEnabled: true, ufoOrbitSpeed: 0.4,
}
```

### Level 8 — "Ligeia Shore" (Titan)

**Design intent:** More debris, pad narrowed, UFO continues. Terrain more complex with tighter navigation corridor.

```javascript
{
  id: 8, name: "Ligeia Shore", environment: 'TITAN',
  gravity: 1.35, drag: 0.0008, wind: 0, gustEnabled: false, gustAmplitude: 0, gustFrequency: 0,
  startingFuel: 700,
  maxDescentSpeed: 45, maxLateralSpeed: 22,
  spawnX: 150, spawnY: 80,
  terrain: [
    { x: 0,   y: 520 }, { x: 80,  y: 490 }, { x: 160, y: 460 }, { x: 220, y: 430 },
    { x: 290, y: 430 }, // decoy zone
    { x: 360, y: 430 },
    { x: 410, y: 410 }, { x: 480, y: 390 },
    { x: 540, y: 390 }, // pad zone
    { x: 610, y: 390 },
    { x: 670, y: 415 }, { x: 760, y: 450 }, { x: 860, y: 475 }, { x: 900, y: 490 },
  ],
  terrainColour: '#D4A843', terrainEdgeColour: '#E8C060', background: '#0D0D1A',
  landingPads: [
    { id: 'pad_a', x: 548, width: 55, y: 390, active: true,  beacon: true  },
    { id: 'pad_b', x: 298, width: 55, y: 430, active: false, beacon: false },
  ],
  debrisPieces: [
    { id: 'd1', shape: 'rock',    x: 140, y: 300, width: 24, height: 20, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 20 },
    { id: 'd2', shape: 'shard',   x: 380, y: 250, width: 10, height: 40, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 8  },
    { id: 'd3', shape: 'boulder', x: 650, y: 310, width: 42, height: 36, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 0  },
    { id: 'd4', shape: 'rock',    x: 280, y: 350, width: 22, height: 18, drift: 12,  driftMin: 200, driftMax: 360, rotation: 45 },
    { id: 'd5', shape: 'rock',    x: 720, y: 270, width: 26, height: 22, drift: 10,  driftMin: 660, driftMax: 780, rotation: 12 },
  ],
  ufoEnabled: true, ufoOrbitSpeed: 0.4,
}
```

### Level 9 — "Titan Rift" (Titan)

**Design intent:** Final Titan level. Jagged terrain, most debris yet, narrow pad at 50px. UFO at full Titan speed.

```javascript
{
  id: 9, name: "Titan Rift", environment: 'TITAN',
  gravity: 1.35, drag: 0.0008, wind: 0, gustEnabled: false, gustAmplitude: 0, gustFrequency: 0,
  startingFuel: 650,
  maxDescentSpeed: 42, maxLateralSpeed: 20,
  spawnX: 750, spawnY: 80,
  terrain: [
    { x: 0,   y: 530 }, { x: 70,  y: 490 }, { x: 130, y: 450 }, { x: 180, y: 415 },
    { x: 240, y: 415 }, // false flat
    { x: 290, y: 415 },
    { x: 340, y: 390 }, { x: 400, y: 365 },
    { x: 450, y: 365 }, // pad zone
    { x: 510, y: 365 },
    { x: 560, y: 390 }, { x: 620, y: 425 }, { x: 690, y: 450 }, { x: 760, y: 420 },
    { x: 820, y: 390 }, { x: 870, y: 410 }, { x: 900, y: 440 },
  ],
  terrainColour: '#D4A843', terrainEdgeColour: '#E8C060', background: '#0D0D1A',
  landingPads: [
    { id: 'pad_a', x: 455, width: 50, y: 365, active: true,  beacon: true  },
    { id: 'pad_b', x: 247, width: 50, y: 415, active: false, beacon: false },
  ],
  debrisPieces: [
    { id: 'd1', shape: 'rock',    x: 120, y: 320, width: 26, height: 22, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 35 },
    { id: 'd2', shape: 'shard',   x: 310, y: 260, width: 10, height: 42, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 12 },
    { id: 'd3', shape: 'boulder', x: 560, y: 290, width: 46, height: 40, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 5  },
    { id: 'd4', shape: 'rock',    x: 740, y: 330, width: 28, height: 22, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 50 },
    { id: 'd5', shape: 'rock',    x: 200, y: 280, width: 22, height: 18, drift: 14,  driftMin: 130, driftMax: 270, rotation: 25 },
    { id: 'd6', shape: 'shard',   x: 620, y: 240, width: 10, height: 40, drift: 11,  driftMin: 570, driftMax: 670, rotation: 18 },
  ],
  ufoEnabled: true, ufoOrbitSpeed: 0.4,
}
```

---

### Level 10 — "Prometheus Plume" (Io)

**Design intent:** Introduce combined wind + debris. Io's first level — moderately hard. Wind leftward, three debris pieces.

```javascript
{
  id: 10, name: "Prometheus Plume", environment: 'IO',
  gravity: 1.80, drag: 0.0001, wind: -2.0, gustEnabled: true, gustAmplitude: 1.5, gustFrequency: 0.6,
  startingFuel: 600,
  maxDescentSpeed: 42, maxLateralSpeed: 18,
  spawnX: 700, spawnY: 80,
  terrain: [
    { x: 0,   y: 515 }, { x: 90,  y: 480 }, { x: 180, y: 450 },
    { x: 250, y: 450 }, // pad zone
    { x: 340, y: 450 },
    { x: 410, y: 430 }, { x: 490, y: 410 }, { x: 570, y: 430 },
    { x: 640, y: 455 }, // false flat
    { x: 720, y: 455 },
    { x: 800, y: 480 }, { x: 900, y: 510 },
  ],
  terrainColour: '#E8C840', terrainEdgeColour: '#F5D855', background: '#0A0E1A',
  landingPads: [
    { id: 'pad_a', x: 258, width: 50, y: 450, active: true,  beacon: true  },
    { id: 'pad_b', x: 648, width: 50, y: 455, active: false, beacon: false },
  ],
  debrisPieces: [
    { id: 'd1', shape: 'rock',    x: 420, y: 300, width: 28, height: 24, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 20 },
    { id: 'd2', shape: 'boulder', x: 600, y: 270, width: 44, height: 38, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 8  },
    { id: 'd3', shape: 'rock',    x: 160, y: 340, width: 24, height: 20, drift: 15,  driftMin: 80,  driftMax: 240, rotation: 40 },
    { id: 'd4', shape: 'shard',   x: 500, y: 220, width: 10, height: 42, drift: 12,  driftMin: 450, driftMax: 560, rotation: 15 },
    { id: 'd5', shape: 'rock',    x: 780, y: 320, width: 26, height: 22, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 55 },
  ],
  ufoEnabled: true, ufoOrbitSpeed: 0.6,
}
```

### Level 11 — "Loki Patera" (Io)

**Design intent:** Strongest wind yet, more debris, 46px pad. Terrain with dramatic central volcanic peak forces player to commit to one side.

```javascript
{
  id: 11, name: "Loki Patera", environment: 'IO',
  gravity: 1.80, drag: 0.0001, wind: +2.5, gustEnabled: true, gustAmplitude: 2.0, gustFrequency: 0.7,
  startingFuel: 600,
  maxDescentSpeed: 39, maxLateralSpeed: 16,
  spawnX: 450, spawnY: 80,
  terrain: [
    { x: 0,   y: 530 }, { x: 80,  y: 505 }, { x: 160, y: 480 },
    { x: 210, y: 480 }, // false flat left
    { x: 260, y: 480 },
    { x: 320, y: 440 }, { x: 400, y: 380 }, { x: 450, y: 350 }, // volcanic peak
    { x: 500, y: 380 }, { x: 580, y: 420 },
    { x: 630, y: 460 }, // active pad zone
    { x: 690, y: 460 },
    { x: 750, y: 490 }, { x: 840, y: 510 }, { x: 900, y: 520 },
  ],
  terrainColour: '#E8C840', terrainEdgeColour: '#F5D855', background: '#0A0E1A',
  landingPads: [
    { id: 'pad_a', x: 637, width: 46, y: 460, active: true,  beacon: true  },
    { id: 'pad_b', x: 216, width: 46, y: 480, active: false, beacon: false },
  ],
  debrisPieces: [
    { id: 'd1', shape: 'rock',    x: 130, y: 360, width: 26, height: 22, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 30 },
    { id: 'd2', shape: 'shard',   x: 350, y: 260, width: 10, height: 44, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 10 },
    { id: 'd3', shape: 'boulder', x: 490, y: 240, width: 46, height: 40, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 0  },
    { id: 'd4', shape: 'rock',    x: 700, y: 330, width: 26, height: 22, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 48 },
    { id: 'd5', shape: 'rock',    x: 250, y: 310, width: 22, height: 18, drift: 16,  driftMin: 160, driftMax: 340, rotation: 22 },
    { id: 'd6', shape: 'shard',   x: 580, y: 200, width: 10, height: 40, drift: 14,  driftMin: 530, driftMax: 640, rotation: 20 },
    { id: 'd7', shape: 'rock',    x: 820, y: 280, width: 24, height: 20, drift: 13,  driftMin: 760, driftMax: 880, rotation: 38 },
  ],
  ufoEnabled: true, ufoOrbitSpeed: 0.6,
}
```

### Level 12 — "Pele's Caldera" (Io)

**Design intent:** The final level. Maximum everything. Strongest wind, most debris, narrowest pad (42px — 10px narrower than legspan). Terrain is maximally hostile. A perfect run feels earned.

```javascript
{
  id: 12, name: "Pele's Caldera", environment: 'IO',
  gravity: 1.80, drag: 0.0001, wind: -3.0, gustEnabled: true, gustAmplitude: 2.5, gustFrequency: 0.8,
  startingFuel: 600,
  maxDescentSpeed: 36, maxLateralSpeed: 14,
  spawnX: 200, spawnY: 80,
  terrain: [
    { x: 0,   y: 540 }, { x: 60,  y: 510 }, { x: 120, y: 470 }, { x: 170, y: 440 },
    { x: 220, y: 440 }, // false flat
    { x: 270, y: 440 },
    { x: 320, y: 410 }, { x: 380, y: 370 }, { x: 430, y: 340 }, // caldera wall
    { x: 480, y: 370 }, { x: 540, y: 400 },
    { x: 590, y: 420 }, // active pad — narrow ledge
    { x: 640, y: 420 },
    { x: 690, y: 400 }, { x: 750, y: 370 }, { x: 800, y: 390 },
    { x: 850, y: 430 }, // far false flat
    { x: 900, y: 450 },
  ],
  terrainColour: '#E8C840', terrainEdgeColour: '#F5D855', background: '#0A0E1A',
  landingPads: [
    { id: 'pad_a', x: 594, width: 42, y: 420, active: true,  beacon: true  },
    { id: 'pad_b', x: 224, width: 42, y: 440, active: false, beacon: false },
    { id: 'pad_c', x: 854, width: 42, y: 430, active: false, beacon: false },
  ],
  debrisPieces: [
    { id: 'd1',  shape: 'rock',    x: 100, y: 370, width: 26, height: 22, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 25 },
    { id: 'd2',  shape: 'shard',   x: 290, y: 280, width: 10, height: 44, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 5  },
    { id: 'd3',  shape: 'boulder', x: 430, y: 240, width: 48, height: 42, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 0  },
    { id: 'd4',  shape: 'shard',   x: 500, y: 190, width: 10, height: 44, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 15 },
    { id: 'd5',  shape: 'rock',    x: 680, y: 290, width: 26, height: 22, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 42 },
    { id: 'd6',  shape: 'boulder', x: 800, y: 310, width: 44, height: 38, drift: 0,   driftMin: 0,   driftMax: 0,   rotation: 12 },
    { id: 'd7',  shape: 'rock',    x: 180, y: 310, width: 22, height: 18, drift: 18,  driftMin: 100, driftMax: 260, rotation: 32 },
    { id: 'd8',  shape: 'shard',   x: 540, y: 220, width: 10, height: 42, drift: 16,  driftMin: 490, driftMax: 600, rotation: 22 },
  ],
  ufoEnabled: true, ufoOrbitSpeed: 0.6,
}
```

---

## 5. Complete Parameter Table

A summary of all per-level parameters for quick reference during implementation and balancing.

| L | Name | Env | Fuel | Descent | Lateral | Wind | Gusts | Drag | Debris | UFO |
|---|------|-----|------|---------|---------|------|-------|------|--------|-----|
| 1 | Sea of Tranquillity | Moon | 1000 | 60 | 30 | 0 | No | 0 | 0 | No |
| 2 | Copernicus Ridge    | Moon | 950  | 57 | 28 | 0 | No | 0 | 0 | No |
| 3 | Tycho Basin         | Moon | 900  | 54 | 26 | 0 | No | 0 | 0 | No |
| 4 | Hellas Planitia     | Mars | 800  | 54 | 24 | +0.8 | No | .0002 | 0 | No |
| 5 | Valles Marineris    | Mars | 800  | 51 | 22 | -1.2 | ±0.6 | .0002 | 0 | No |
| 6 | Olympus Approach    | Mars | 750  | 48 | 20 | +1.5 | ±1.0 | .0002 | 0 | No |
| 7 | Kraken Mare         | Titan| 700  | 48 | 24 | 0 | No | .0008 | 3 | Yes 0.4 |
| 8 | Ligeia Shore        | Titan| 700  | 45 | 22 | 0 | No | .0008 | 5 | Yes 0.4 |
| 9 | Titan Rift          | Titan| 650  | 42 | 20 | 0 | No | .0008 | 6 | Yes 0.4 |
| 10| Prometheus Plume    | Io   | 600  | 42 | 18 | -2.0 | ±1.5 | .0001 | 5 | Yes 0.6 |
| 11| Loki Patera         | Io   | 600  | 39 | 16 | +2.5 | ±2.0 | .0001 | 7 | Yes 0.6 |
| 12| Pele's Caldera      | Io   | 600  | 36 | 14 | -3.0 | ±2.5 | .0001 | 8 | Yes 0.6 |

---

## 6. Level Design Validation Checklist

Before finalising any level definition, verify:

- [ ] All terrain vertices strictly left-to-right, x=0 to x=900
- [ ] All Y values in range [200, 580]
- [ ] Landing pad Y matches terrain Y at that X range exactly
- [ ] Active pad has at least 120px clear airspace directly above
- [ ] Spawn point has at least 200px open airspace below
- [ ] Active pad reachable from spawn with level's startingFuel (playtested)
- [ ] All debris at least 80px from active landing pad
- [ ] All debris has at least 60px clear flight path around it
- [ ] No debris within 150px below spawn point
- [ ] All debris at least 20px above terrain surface
- [ ] First and last terrain vertex Y values within 50px of each other (wrap continuity)
- [ ] Decoy pads (if any) have `active: false` and `beacon: false`

---

## 7. Resolved Decisions

| # | Question | Decision | Rationale |
|---|----------|----------|-----------|
| 1 | Should level names be real solar system locations? | Yes — authentic names where possible | Real names (Valles Marineris, Kraken Mare, Pele's Caldera) give the game grounding and make each level feel like a real place. Players who know their planetary science get an extra layer of resonance. |
| 2 | Should debris positions be pixel-exact or approximate? | Pixel-exact — final positions authored in this spec | Approximate positions invite drift during implementation. Exact coordinates mean the authored level is exactly what ships. Balancing adjustments are made by updating this spec. |
| 3 | Should Titan levels have wind? | No — drag only, no wind | Wind and drag together would be too much for Titan. Drag alone gives Titan its distinctive heavy feel. Wind returns on Io with debris already established. |
| 4 | Should Io starting fuel decrease across the three levels? | No — constant at 600 | Io levels 10–12 already ramp difficulty heavily via wind strength, debris count, pad width, and landing thresholds. Reducing fuel further would make the levels feel impossibly constrained. 600 is tight enough. |

---

*End of SPEC-12-LevelDefinitions*
