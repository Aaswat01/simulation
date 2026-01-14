# Leaflet OSM Intelligent Navigation System

## Project Overview

This repository implements a **full‑scale, browser‑based intelligent navigation and driving‑assistance system** built on top of **OpenStreetMap (OSM)** and **Leaflet.js**, intentionally designed to **mirror and extend Google‑Maps‑style navigation behavior** while remaining **fully open, inspectable, and API‑cost‑free**.

Unlike demo routing projects, this system behaves as a **real navigation engine**:

* The *driver’s live GPS location* is the authoritative state
* Instructions adapt dynamically to movement, deviation, and speed
* Routes are not static polylines but **stateful navigation graphs**

The codebase is written in **pure HTML, CSS, and vanilla JavaScript** with zero frameworks. This is a deliberate architectural choice to:

1. Enable deep understanding of every internal mechanism
2. Allow deployment on low‑resource or embedded devices
3. Avoid abstraction leakage common in mapping SDKs

This README is written as **knowledge‑transfer documentation**, not a marketing overview. Every major subsystem is explained down to **data structures, math, control flow, and failure handling**.

---

## System Goals and Non‑Goals

### Goals

* Google‑Maps‑like navigation UX using only open data
* Deterministic, debuggable routing behavior
* Location‑first logic (no blind step playback)
* Multi‑destination trip management (A → B → C)
* Driver‑centric map rotation and recentering
* Extensible AI‑style advisory layer

### Non‑Goals

* No proprietary Google APIs
* No heavy frontend frameworks
* No black‑box SDK routing engines

---

## Repository Structure

```
Browser
 ├── dashboard.html
 │    ├── UI (HTML + CSS)
 │    ├── Map initialization
 │    ├── Navigation logic
 │    ├── AI assistance layer
 │
 ├── leaflet-rotate-src.js
 │    └── Map & marker rotation engine
 │
 ├── leaflet.polylineDecorator.js
 │    └── Direction arrows & path symbols
 │
 └── External APIs
      ├── OSRM routing
      ├── Nominatim search
      ├── Elevation data
```

The system is intentionally **flat‑structured** to simplify onboarding and debugging.

---

## External Dependencies (Explicit)

| Component         | Purpose            | Link                                                                                                           |
| ----------------- | ------------------ | -------------------------------------------------------------------------------------------------------------- |
| Leaflet.js        | Map engine         | "https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"                                                             |
| OSRM              | Routing backend    | [https://project-osrm.org](https://project-osrm.org)                                                           |
| OpenStreetMap     | Map data           | [https://www.openstreetmap.org](https://www.openstreetmap.org)                                                 |
| PolylineDecorator | Direction arrows   | [leaflet.polylineDecorator.js](https://github.com/bbecquet/Leaflet.PolylineDecorator)                          |
| Leaflet‑Rotate    | Map rotation       | [leaflet-rotate-src.js](https://github.com/Raruto/leaflet-rotate)                                              |
| Nominatim         | Search & geocoding | (https://nominatim.openstreetmap.org/reverse?format=jsonv2&lat=${latlng.lat}&lon=${latlng.lng})                |
| Routing           | Routing Leaflet    | [https://unpkg.com/leaflet-routing-machine@latest/dist/leaflet-routing-machine.js]                             |
---

## Application Boot Sequence (Step‑by‑Step)

1. **HTML & CSS load**

   * UI containers rendered
   * All controls positioned but inactive

2. **Leaflet core initialization**

   * Map instance created with rotation enabled
   * Custom panes registered

3. **Tile layers attached**

   * Google tiles, OSM, Satellite, Night mode
   * Layer switcher initialized

4. **Marker system prepared**

   * Driver dot icon (idle)
   * Navigation arrow icon (active)

5. **Global state initialized**

   * Routing arrays
   * Speech cooldown trackers
   * API request queue

6. **Location watcher activated**

   * Driver position becomes system truth

---
File Breakdown

## 1. index.html

This is the single‑page application entry point. It contains:

*Complete UI layout

*All runtime JavaScript logic

* Map initialization

* Navigation state machine

Why single‑file?

*Zero build system

*Easy offline hosting

* Simple deployment on embedded systems

## 2. leaflet.polylineDecorator.js

* Responsible for visual route direction indicators.

* Key capabilities:

* Arrow heads aligned with route direction

* Dash / symbol repetition along polyline

* Heading calculation per segment

Core Math

Segment Heading Calculation
```
heading = atan2(Δy, Δx) × 180/π
```
Each arrow is rotated to match the bearing of the route segment it belongs to.

InterpolationArrows are placed at fixed distance ratios along the polyline using linear interpolation between segment endpoints.

## 3. leaflet-rotate-src.js

Extends Leaflet to support true map rotation, not just marker rotation.

Features:

*Rotate map canvas

*Rotate markers relative to map or world

*Maintain popup & tooltip alignment

*Bearing‑aware coordinate transforms

Rotation Math

2D Rotation Matrix:
```
[x']   [ cosθ  -sinθ ] [x]
[y'] = [ sinθ   cosθ ] [y]
```
Applied around a pivot point (map center) to keep navigation heading‑up.

---
## Map Engine Deep Dive (Leaflet)

Leaflet is used as a **low‑level rendering engine**, not as a routing solution.

### Why Leaflet

* Predictable rendering pipeline
* Direct access to map internals
* Plugin‑friendly architecture

### Map Configuration

```js
L.map('map', {
  rotate: true,
  touchRotate: true,
  inertia: true
});
```

Rotation is enabled at the **map level**, not just markers. This distinction is critical for navigation realism.

---

## Tile Layer System

Multiple tile sources are supported concurrently:

| Layer      | Purpose               |
| ---------- | --------------------- |
| Google Map | Familiar road styling |
| OSM        | Pure open data        |
| Satellite  | Visual context        |
| Night Mode | Low‑light driving     |

Switching layers **does not reset** navigation, routes, or steps.

---

## Marker Architecture

### Driver Marker States

| State  | Icon     | Description          |
| ------ | -------- | -------------------- |
| Idle   | Blue Dot | No navigation active |
| Active | Arrow    | Navigation running   |

The arrow icon rotation is driven by **real bearing**, not map orientation.

---

## Routing Engine (OSRM)

### Primary Endpoint

```
https://router.project-osrm.org/route/v1/driving/{lon1},{lat1};{lon2},{lat2}
```

### Returned Data

* Encoded geometry (polyline)
* Ordered step instructions
* Distance per step
* Duration per step

The system **does not trust OSRM blindly**. All step progression is validated against live GPS.

---

## Safe Fetch & API Queue (Critical System)

Public APIs enforce rate limits and can fail unpredictably.

### Problem

* Multiple rapid searches
* Rerouting bursts
* Elevation lookups

### Solution

A **bounded asynchronous queue**:

```js
MAX_REQUESTS = 6
REQUEST_DELAY = 120ms
```

#### Behavior

* Requests are queued
* Retries on 429 / 504
* Hard cancellation via sequence tokens

This prevents corrupted routing state and UI desynchronization.

---

## Navigation Lifecycle (State Machine)

### 1. Destination Discovery

User can:

* Search via text
* Tap map in Select mode

Multiple candidate results are returned using Nominatim + auxiliary APIs.

---

### 2. Route Preview Phase

* Temporary polyline rendered
* No arrow decoration
* No speech

Purpose: **visual confirmation without commitment**.

---

### 3. Approval Panel

User must explicitly confirm:

* Destination
* Distance
* Estimated duration

No navigation begins without approval.

---

### 4. Active Navigation Phase

On approval:

* Arrow polyline decorator enabled
* Driver marker switches to arrow
* Map enters heading‑up mode
* Step engine activated

---

## Polyline Decorator Internals (Math)

### Segment Heading Calculation

For each polyline segment:

```
heading = atan2(Δy, Δx) × 180/π
```

Arrows are rotated to align with segment heading.

### Arrow Placement

* Total path length computed
* Repetition interval applied
* Linear interpolation places symbols

This ensures arrows scale correctly at all zoom levels.

---

## Map Rotation Engine (Leaflet‑Rotate)

This subsystem performs **true map rotation**, not CSS tricks.

### Core Math

2D rotation matrix:

```
[x'] = [ cosθ  −sinθ ] [x]
[y']   [ sinθ   cosθ ] [y]
```

Applied around the map’s pivot point.

### Why Rotate the Map

* Human driving perception is heading‑up
* Reduces cognitive load
* Matches automotive navigation UX

---

## Step Management Engine

Each route contains an ordered step list.

### Step Activation Logic

```text
if distance(driver, step_end) < 30m:
    step.completed = true
    activate next step
```

Completed steps are removed from UI to prevent clutter.

---

## Distance & Position Math

### Haversine Distance

Used for:

* Step arrival
* Remaining distance
* Deviation detection

```
d = 2R · asin(√(sin²(Δφ/2) + cosφ1·cosφ2·sin²(Δλ/2)))
```

---

## Automatic Rerouting

Triggered when:

* Driver deviates beyond threshold from polyline
* Cooldown elapsed

```js
REROUTE_COOLDOWN = 6000ms
```

Prevents oscillation and API abuse.

---

## Map Recentering & Manual Override

### Auto Mode

* Driver always centered
* Map rotates with heading

### Manual Interaction

* User drags or rotates map
* Auto resumes after inactivity timeout

```js
manualRotationTimeout = 4000ms
```

---

## Multi‑Stop Routing Engine

Supports chained destinations:

```
A → B → C → D
```

Each stop:

* Has its own route box
* Activates sequentially
* Preserves previous state

---

## AI Tip & Advisory System

### Inputs

* Speed
* Speed limit
* Road type
* Slope (elevation)
* Step distance

### Outputs

* Turn announcements
* Overspeed warnings
* Eco‑driving tips

### Cooldown Model

```js
ECO_COOLDOWN = 20s
STEP_REMINDER = 20s
```

Ensures relevance without spam.

---

## Speed Limit Inference

Speed limits are derived from:

* OSM road metadata
* Regional defaults

Used for both display and warnings.

---

## Localization & Translation

When crossing borders:

* Step text auto‑translated
* Units adapted
* Speech updated

---

## Destination Arrival Handling

On arrival:

* AI announces completion
* Route box closes
* Next route auto‑activates (if present)

---

## Failure Handling & Edge Cases

* API timeouts
* Partial route responses
* GPS jitter
* Step misalignment

All handled defensively with state checks.

---
## License & Attribution

OpenStreetMap contributors

Leaflet.js

OSRM Project
