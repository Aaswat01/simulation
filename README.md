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

## 📈 ASCII CALL-FLOW DIAGRAM
```
User Interaction
   │
   ├─ Text Search / Voice Search
   │      │
   │      ├─ getSearchResults()
   │      │     ├─ Photon API
   │      │     ├─ Nominatim API
   │      │     └─ Wikipedia API
   │      │
   │      └─ openPreviewTo(latlng)
   │             ├─ reverseGeocode()
   │             ├─ speak("Destination set")
   │             └─ buildPreviewRoute()
   │                    └─ OSRM.route()
   │
   ├─ Map Tap (Select Mode)
   │      │
   │      └─ openPreviewTo(latlng)
   │
   ├─ Approval Panel → START
   │      │
   │      └─ approvalStartBtn.onclick
   │             ├─ clearPreview()
   │             ├─ createRoute()
   │             │     └─ OSRM_ROUTER.route()
   │             ├─ enable arrow decorators
   │             ├─ autoRotation = true
   │             └─ activate step engine
   │
GPS Position Update (watchPosition)
   │
   ├─ calculateGPSSpeed()
   ├─ updateSpeedLimit()
   ├─ updateNavArrow()
   │      └─ getPathBearing()
   ├─ rotateTowardRoute()
   ├─ smartStepAdvancement()
   ├─ checkUpcomingSteps()
   ├─ adaptiveAIDashboard()
   │      ├─ getRoadType()
   │      ├─ predictElevationAhead()
   │      ├─ ecoAdvisor()
   │      └─ speak() / aiBoxShow()
   │
Deviation Detection
   │
   └─ getNearestRouteDistance()
          └─ triggerReroute()
                 └─ createRoute() (again)

```
---
## File Breakdown

1. index.html

This is the single‑page application entry point. It contains:

* Complete UI layout

* All runtime JavaScript logic

* Map initialization

* Navigation state machine

Why single‑file?

* Zero build system

* Easy offline hosting

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

* Rotate map canvas

* Rotate markers relative to map or world

* Maintain popup & tooltip alignment

* Bearing‑aware coordinate transforms

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

Functions / Objects :
```
const googleMap = L.tileLayer(...)
const osmMap = L.tileLayer(...)
const nightMap = L.tileLayer(...)
L.control.layers(...)
```

Logic :

* Each tile layer is stateless
 
* Switching layers does not reset navigation

* Routes, markers, rotation remain untouched
---

## Marker Architecture

### Driver Marker States

| State  | Icon     | Description          |
| ------ | -------- | -------------------- |
| Idle   | Blue Dot | No navigation active |
| Active | Arrow    | Navigation running   |

The arrow icon rotation is driven by **real bearing**, not map orientation.

Functions / Objects:
```
const dotIcon = L.divIcon({...})
const navArrowIcon = L.divIcon({...})
```
Switching Logic : 
```
function updateNavArrow(latlng) {
  if (!activeRouteIndex) {
    userMarker.setIcon(dotIcon);
    return;
  }
  userMarker.setIcon(navArrowIcon);
}
```
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

Functions:

```
async function safeFetch(url, retries)
async function processQueue()
````
Globals:
```
MAX_REQUESTS = 6
REQUEST_DELAY = 120ms
activeRequests
requestQueue
````

What it prevents:

* OSRM 429 errors

* Elevation API overload

* Search spam crashes
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

Code Location

Core Functions: 
```
function buildPreviewRoute(startLL, endLL, labelText)
function openPreviewTo(latlng, label)
function clearPreview()
```

Flow:

* User clicks/searches

* openPreviewTo() resolves address

* buildPreviewRoute() fetches OSRM

* Preview polyline rendered

Approval panel shown
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
  
Function Trigger:
```
approvalStartBtn.addEventListener("click", ...)
```
----
## FUNCTION CALL GRAPH
```
UI Layer
│
├─ openPreviewTo()
│    ├─ reverseGeocode()
│    ├─ speak()
│    └─ buildPreviewRoute()
│          └─ OSRM_ROUTER.route()
│
├─ approvalStartBtn.onclick
│    ├─ clearPreview()
│    ├─ createRoute()
│    └─ rotateTowardRoute()
│
GPS Core Loop
│
├─ calculateGPSSpeed()
├─ updateSpeedLimit()
├─ updateNavArrow()
│    └─ getPathBearing()
│          └─ atan2()
├─ rotateTowardRoute()
├─ smartStepAdvancement()
│    ├─ findCurrentStepByLocation()
│    ├─ updateStepUI()
│    └─ speak()
├─ checkUpcomingSteps()
└─ adaptiveAIDashboard()
     ├─ getRoadType()
     ├─ predictElevationAhead()
     │     └─ getElevations()
     ├─ ecoAdvisor()
     └─ maybeSpeakEco()

Deviation Handling
│
└─ getNearestRouteDistance()
     └─ distanceToSegment()
          └─ vector projection math
```
----
## STATE MACHINE DIAGRAM
```
[ IDLE ]
   │
   ├─ Search / Map Tap
   ▼
[ PREVIEW ]
   │
   ├─ Cancel ─────────────► [ IDLE ]
   │
   └─ Approve
        │
        ▼
[ NAVIGATING ]
        │
        ├─ GPS Update
        │     ├─ Step Advancement
        │     ├─ AI Advice
        │     └─ Map Rotation
        │
        ├─ Off-Route
        │     └─ [ REROUTING ]
        │              │
        │              └─ Back to [ NAVIGATING ]
        │
        └─ Final Step
               │
               ▼
      [ DESTINATION REACHED ]
               │
               ├─ Next Route Exists ─► [ NAVIGATING ]
               └─ No More Routes ────► [ IDLE ]
```
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

Core Function:
```
function getPathBearing(latlng, route)
```
Internal Math
```
const angle = Math.atan2(p2.lng - p1.lng, p2.lat - p1.lat) * 180 / Math.PI
```

Purpose: 

* Computes forward direction along route

* Uses look-ahead distance (20–30m) for smooth rotation

* Avoids jitter from GPS noise

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

Core Functions:
* findCurrentStepByLocation()
* smartStepAdvancement()
* updateStepUI()

Step Detection Logic:
```
distance(driver, step_end) < STEP_ARRIVAL_THRESHOLD
```
Why this matters

Steps are location-validated, not time-based.

---
## SEQUENCE DIAGRAM
```
Time →
┌────────────┐
│ GPS Update │
└─────┬──────┘
      │
      ├─ calculateGPSSpeed()
      │
      ├─ updateSpeedLimit()
      │
      ├─ updateNavArrow()
      │
      ├─ smartStepAdvancement()
      │     ├─ step reached?
      │     └─ speak("Turn instruction")
      │
      ├─ adaptiveAIDashboard()
      │     ├─ predictElevationAhead()
      │     ├─ ecoAdvisor()
      │     └─ maybeSpeakEco()
      │
      ├─ UI Update
      │     ├─ Speed
      │     ├─ Gear
      │     ├─ Throttle
      │     └─ AI Tip Box
      │
      └─ Cooldown Guards
            ├─ SPEECH.WARNING_HOLD
            ├─ ECO_COOLDOWN
            └─ STEP_REMINDER
```
---

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

Core Functions:
```
getNearestRouteDistance()
distanceToSegment()
```
Geometry Used:

* Point-to-segment projection
* Sliding window optimization (±100 points)

Reroute Trigger:
```
if (distance > threshold && cooldown expired)
```

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

Inputs → Functions:
| Input          | Function                  |
| -------------- | ------------------------- |
| Speed          | `calculateGPSSpeed()`     |
| Road Type      | `getRoadType()`           |
| Elevation      | `getElevations()`         |
| Slope          | `predictElevationAhead()` |
| Distance Ahead | `LOOKAHEAD_DISTANCE`      |

Decision Engine:
```
ecoAdvisor(distAhead, slope)
```

Returns:
```
{ speed, gear, throttle, tip }
```

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

Function:
```
async function updateSpeedLimit(latlng)
```
Data Source

* Overpass API
* OSM maxspeed tag

Special Handling:

* Arabic numerals → Latin
* mph → km/h conversion
* Sanity bounds (10–200 km/h)
  
Speed limits are derived from:

* OSM road metadata
* Regional defaults

Used for both display and warnings.

---

## Localization & Translation

Speech Engine:
* function speak(text, kind)
* function refreshAIBox()

Translation Trigger

Occurs when:

* Nominatim returns non-local language
* Step names differ from UI locale
  
When crossing borders:

* Step text auto‑translated
* Units adapted
* Speech updated

---

## Destination Arrival Handling

Logic:
```
if (latlng.distanceTo(lastStep) < 20m)
```
Function:
```
cleanupRoute(route)
On arrival:
```

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

-----
## 🚨 FAILURE FLOW DIAGRAM
```
GPS Permission Denied
   │
   └─ Enable Virtual GPS
        └─ updateVirtualGPS()

OSRM API Failure
   │
   ├─ Preview Phase
   │     └─ Allow START anyway
   │
   └─ Active Navigation
         └─ Retry with cooldown
               └─ Keep last route

Elevation API Failure
   │
   └─ Default slope = 0
        └─ Disable slope-based tips

Speed Limit API Failure
   │
   └─ Keep last known limit
        └─ UI shows "--"

Speech Engine Failure
   │
   └─ AI Box remains active
```

---

| Global             | Purpose              |
| ------------------ | -------------------- |
| `activeRouteIndex` | Current route        |
| `currentStepIndex` | Active step          |
| `announcedSteps`   | Speech deduplication |
| `autoRotation`     | Route-up mode        |
| `follow`           | Auto-centering       |
| `SPEECH.*`         | Speech cooldowns     |
| `lastSpeedLimit`   | Cached speed         |



## License & Attribution

* OpenStreetMap contributors

* Leaflet.js

* OSRM Project
