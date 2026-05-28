# Driving Simulator — Project Outline

## Architecture Overview

```
Driving-Simulator/
└── index.html  (1120 lines)
     │
     ├── 1. Config Panel (L216-240)
     │    ├── accelInput  — acceleration a_accel (m/s²)
     │    ├── brakeInput  — braking a_brake (m/s²)
     │    └── startBtn    — triggers startGame()
     │
     ├── 2. Game Canvas (L243-245)
     │    └── <canvas id="gameCanvas">  960×440 px
     │         │
     │         ├── 2a. Sky (L382-390)
     │         │    └── linear gradient: blue → light blue → pale
     │         │
     │         ├── 2b. Scenery (L341-428)
     │         │    ├── mountains[]  — 30 triangles, parallax rate 0.6
     │         │    ├── trees[]      — 60 sprites (trunk + circle), rate 0.95
     │         │    └── buildings[]  — 25 rectangles with windows, rate 0.5
     │         │
     │         ├── 2c. Road (L429-439)
     │         │    ├── asphalt surface (#3d3d3d)
     │         │    ├── top border (#555)
     │         │    └── dashed lane markings (white, 30px/60px cycle)
     │         │
     │         ├── 2d. Car (L440-543)
     │         │    ├── shadow ellipse
     │         │    ├── two wheels (rotating spokes, angle = s/14)
     │         │    ├── body (red polygon, rounded shape)
     │         │    ├── cabin roof (curved top)
     │         │    ├── windows (light blue)
     │         │    ├── headlight (yellow)
     │         │    └── taillight (red)
     │         │
     │         ├── 2e. HUD (L545-594)
     │         │    ├── speed v (m/s, blue/red for reverse)
     │         │    ├── time t (s, white)
     │         │    ├── displacement s (m, green/red, rounded integer)
     │         │    └── acceleration a (m/s², orange/red, rounded to 1 decimal)
     │         │
     │         └── 2f. Pedal Indicator (L596-623)
     │              ├── J brake pedal (red when pressed)
     │              └── K accel pedal (green when pressed)
     │
     ├── 3. V-T Chart (L247-264)
     │    └── <canvas id="chartCanvas">  960×340 px
     │         │
     │         ├── 3a. Axes & Grid (drawChart: L702-799)
     │         │    ├── v-axis (velocity) with arrowhead
     │         │    ├── t-axis at v=0 with arrowhead
     │         │    └── auto-scaled grid lines (nice numbers)
     │         │
     │         ├── 3b. Data Visualization (drawChart: L800-858)
     │         │    ├── blue fill area (positive v, displacement forward)
     │         │    ├── red fill area (negative v, displacement backward)
     │         │    ├── fillSignedArea() — zero-crossing interpolation (L814-855)
     │         │    ├── v-t curve (#1a56db, 2.5px)
     │         │    ├── data dots (subsampled for performance)
     │         │    ├── current point (red circle, white border)
     │         │    ├── slope annotation (dashed red line + k label)
     │         │    └── signed area labels: +pos (blue), −neg (red), net total (dark)
     │         │
     │         └── 3c. Tooltip (L920-977)
     │              └── nearest-point hover: t + v display
     │
     ├── 4. Physics Engine (L639-649)
     │    ├── updatePhysics(dt)
     │    │    ├── if !simStarted → return (frozen until first K press)
     │    │    ├── a = +cfgAccel  (K pressed)
     │    │    ├── a = -cfgBrake  (J pressed)
     │    │    ├── a = 0          (neither)
     │    │    ├── v += a * dt
     │    │    ├── s += (v0+v)/2 * dt  (trapezoidal)
     │    │    ├── t += dt
     │    │    └── bgOffset += v * PX_PER_M * dt
     │    │
     │    └── Constants (L282-286)
     │         ├── CAR_X = 200           (fixed horizontal position)
     │         ├── ROAD_Y = 320          (road surface starts here)
     │         └── PX_PER_M = 120        (pixels per meter scale)
     │
     ├── 5. Input System (L1090-1103)
     │    ├── keydown → keys['j'|'k'] = true
     │    ├── first K press → simStarted = true (unfreezes physics)
     │    └── keyup   → keys['j'|'k'] = false
     │
     ├── 6. Game Loop (L1000-1021)
     │    ├── gameLoop(now)
     │    │    ├── dt clamping (0.016 ~ 0.1 s)
     │    │    ├── updatePhysics(dt)       [when not paused]
     │    │    ├── vtData sampling @ ~10 Hz [when simStarted & not paused]
     │    │    ├── renderGame()
     │    │    └── requestAnimationFrame(gameLoop)
     │    │
     │    └── State Management (L315-336)
     │         ├── running: boolean
     │         ├── paused:  boolean
     │         ├── simStarted: boolean   (starts on first K press, gates physics)
     │         ├── t, v, s, a:  number  (physics state)
     │         ├── lastTime, lastSample:  number
     │         └── vtData:  {t, v}[]     (chart data buffer)
     │
     ├── 7. Control Buttons & Events (L267-275, L1027-1115)
     │    ├── pauseBtn  → togglePause()
     │    ├── stopBtn   → stopGame()
     │    ├── startBtn  → startGame()
     │    └── statusText
     │
     └── 8. Styling (L7-208)
          ├── dark theme (#1a1a2e background)
          ├── config panel (#16213e, rounded corners)
          └── key hints (color-coded borders)
```

## Module Dependency Graph

```
                    ┌──────────┐
                    │  index   │
                    │  (root)  │
                    └────┬─────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
   ┌─────────┐    ┌──────────┐    ┌──────────┐
   │ Config  │    │  Game    │    │  V-T     │
   │ Panel   │    │  Canvas  │    │  Chart   │
   └────┬────┘    └────┬─────┘    └────┬─────┘
        │               │               │
        ▼               │               │
   startGame() ──────── ┤               │
        │               │               │
        ├──► Physics ◄─ ┤               │
        │    Engine     │               │
        │       │       │               │
        │       ▼       │               │
        │    Input ─────┤               │
        │    System     │               │
        │       │       │               │
        │       ▼       │               │
        ├──► Game Loop──┤               │
        │       │       │               │
        │       ├──► Sky                │
        │       ├──► Scenery            │
        │       ├──► Road               │
        │       ├──► Car                │
        │       ├──► HUD                │
        │       └──► Pedals             │
        │                               │
        └────── vtData ───────────────► drawChart()
                                        │
                                        ├──► Axes/Grid
                                        ├──► Area Fill
                                        ├──► v-t Curve
                                        ├──► Slope Line
                                        └──► Tooltip
```

## Key Relationships

| Module         | Reads From                          | Writes To                             |
| -------------- | ----------------------------------- | ------------------------------------- |
| **Physics**    | `keys[]`, `cfgAccel`, `cfgBrake`    | `v`, `s`, `t`, `a`, `bgOffset`        |
| **Game Loop**  | `running`, `paused`                 | calls Physics, Render, Chart sampling |
| **Car Render** | `s` (displacement), `v` (direction) | —                                     |
| **Scenery**    | `bgOffset`                          | —                                     |
| **V-T Chart**  | `vtData[]`, `a`, `s`                | —                                     |
| **HUD**        | `v`, `t`, `s`, `a`                  | —                                     |
| **Input**      | keyboard events                     | `keys[]`                              |

## Data Flow

```
                    ┌─────────────┐
                    │  Keyboard   │
                    │  (J / K)    │
                    └──────┬──────┘
                           │ keydown / keyup
                           ▼
                    ┌─────────────┐
                    │    keys[]   │
                    │  {j, k}     │
                    └──────┬──────┘
                           │ read by Physics
                           ▼
              ┌────────────────────────┐
              │    updatePhysics(dt)    │
              │                        │
              │  a = +cfgAccel (K)     │
              │  a = -cfgBrake (J)     │
              │  a = 0        (none)   │
              │                        │
              │  v += a * dt           │
              │  s += (v0+v)/2 * dt    │
              │  t += dt               │
              │  bgOffset += v*PPM*dt  │
              └────────┬───────────────┘
                       │
         ┌─────────────┼──────────────┐
         ▼             ▼              ▼
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │ v,s,t,a  │  │ vtData[] │  │bgOffset  │
   │ (physics │  │ (chart   │  │(scenery  │
   │  state)  │  │  buffer) │  │  scroll) │
   └────┬─────┘  └────┬─────┘  └────┬─────┘
        │             │             │
        ▼             ▼             ▼
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │   HUD    │  │ V-T Chart│  │ Scenery  │
   │ speed    │  │ curve    │  │mountains │
   │ time     │  │ area     │  │ trees    │
   │ displ.   │  │ slope    │  │ buildings│
   │ accel.   │  │ tooltip  │  │ road     │
   └──────────┘  └──────────┘  └──────────┘
                       
   ┌──────────────────────────────────────┐
   │             renderGame()             │
   │  Sky → Scenery → Road → Car → HUD   │
   │              → Pedals                │
   └──────────────────────────────────────┘
```

### Sampling Strategy

```
requestAnimationFrame (~60 Hz)
        │
        ├──► updatePhysics(dt)     [every frame, when not paused]
        │
        └──► vtData sampling       [throttled to ~10 Hz, L1013]
             │
             └──► drawChart()      [only when new sample added]

### Key Additions Since v1
- `simStarted` flag — physics frozen until first K (accelerator) press
- Signed area fill — `fillSignedArea()` with zero-crossing interpolation
- `computeSignedAreas()` — returns `{pos, neg, net}` via trapezoidal rule with crossing detection
- Three-line area labels — positive (blue), negative (red), net (dark), all rounded to integer
```
