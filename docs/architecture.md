# London Live 3D Map — Architecture Document

> **Status:** Revised (Post-Review Pass 1)  
> **Version:** 2.0  
> **Date:** 2026-06-18  
> **Author:** Lead Architect  
> **Audience:** Engineers implementing the system  
> **Changes from v1:** All Critical and High severity issues from 3 independent reviews resolved. Major changes: Draco-compressed glTF buildings, backend aggregator with SSE, mutable refs for vehicle animation, entity resolution layer, unified transport schema, deployment strategy, inference engine re-scope.

---

## 1. System Context

### 1.1 What the System Does

Zone One is a real-time 3D visualization of London showing live transport across six layers: Tube trains, buses, National Rail trains, aircraft, ships, and river boats. Transport without GPS (buses, trains) is inferred from arrival/departure data and animated along actual road/rail geometry. The system renders a dark, moody 3D cityscape with neon-glowing transport lines and color-coded vehicle markers.

### 1.2 External Systems

| System | Purpose | Protocol | Auth |
|--------|---------|----------|------|
| TfL API | Tube, Bus, River, Jamcams | HTTPS REST (JSON) | API key (header) |
| Darwin DBRS | National Rail departures | HTTPS REST (JSON) | API key (header) |
| ADSBx v2 | Aircraft positions | HTTPS REST (JSON) | API key (header) |
| AIS Hub | Vessel positions | HTTPS REST (JSON/XML) | Username (query param) |
| Overture Maps | 3D building footprints | S3 (Parquet tiles) | None |
| Overpass API | Road/rail/waterway geometry | HTTPS REST (XML/JSON) | None |

### 1.3 Users

- **Primary:** Transit enthusiasts, urban planners, London residents browsing the live map
- **Interaction model:** Single-page web application, desktop-first, mouse-driven 3D navigation

### 1.4 Major Data Flows

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│   TfL    │    │ Darwin   │    │  ADSBx   │    │  AIS     │
│   API    │    │   DBRS   │    │   v2     │    │  Hub     │
└────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘
     │               │               │               │
     ▼               ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Backend Aggregator (Express)                   │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │  TfL     │  │ Darwin   │  │  ADSBx   │  │  AIS     │       │
│  │ Adapter  │  │ Adapter  │  │ Adapter  │  │ Adapter  │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
│       │              │              │              │            │
│       └──────────────┴──────────────┴──────────────┘            │
│                            │                                    │
│                   ┌────────▼────────┐                           │
│                   │  Vehicle State  │                           │
│                   │  Engine         │                           │
│                   │  (tracking,     │                           │
│                   │   inference,    │                           │
│                   │   entity res.)  │                           │
│                   └────────┬────────┘                           │
│                            │                                    │
│  ┌─────────────────────────▼────────────────────────────────┐  │
│  │                    SSE Broadcast Channel                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  GET /api/sse/transport      (SSE stream — all vehicles)        │
│  GET /api/sse/jamcams        (SSE stream — jamcams)            │
│  GET /api/map/buildings      (static Draco glTF)                │
│  GET /api/map/geometry       (static GeoJSON)                   │
│  GET /api/search?q=...       (vehicle search)                   │
│  GET /api/layers             (layer status + toggle)            │
└─────────────────────────────────────────────────────────────────┘
                             │ SSE / fetch()
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Frontend (Vite + React)                       │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Mutable Refs (simulation state) ← NOT Zustand            │   │
│  │  - vehicles: MutableVehicle[] (updated each frame)        │   │
│  │  - No React re-render on position changes                 │   │
│  └──────────────────────────────────────────────────────────┘   │
│           │                              │                       │
│  ┌────────▼──────────┐      ┌───────────▼──────────┐           │
│  │  R3F 3D Scene     │      │  UI Overlay (HTML)   │           │
│  │  - Buildings      │      │  - Loading screen    │           │
│  │  - Thames water   │      │  - Tooltips          │           │
│  │  - Neon lines     │      │  - Jamcam overlay    │           │
│  │  - Vehicle markers│      │  - Layer toggles     │           │
│  │  - Street lamps   │      │  - Search/filter     │           │
│  └───────────────────┘      └──────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

**Key changes from v1:**
- Backend is an **Aggregator**, not a proxy. It polls APIs on its own schedule, maintains vehicle state, and broadcasts via SSE.
- Frontend receives **render-ready snapshots** — no position calculation on the client.
- Vehicle positions stored in **mutable refs**, not Zustand. React only re-renders for UI state (tooltips, layer toggles, search results).
- Buildings delivered as **Draco-compressed glTF**, not GeoJSON.

---

## 2. Architecture Principles

### 2.1 Key Principles

1. **Dummy-first development** — All data endpoints serve realistic dummy data during development. Real API integration is a drop-in replacement at the end.

2. **Geometry-constrained animation** — Vehicles never move through free space. Bus and train positions are always projected onto the nearest point of the road/rail geometry array.

3. **Instanced rendering** — All vehicles of the same type share a single `InstancedMesh`. Buildings are Draco-compressed glTF loaded via `useGLTF` from Drei.

4. **Simulation state on refs** — Vehicle positions are stored in mutable refs, NOT in React state. The renderer reads refs directly in `useFrame`. React only re-renders for UI interactions (hover, click, filter).

5. **Backend aggregation, not proxy** — The backend polls APIs on its own schedule, maintains the canonical vehicle state, and broadcasts to all clients. One poll serves all users.

6. **SSE for real-time updates** — Server-Sent Events push vehicle snapshots to the frontend. The frontend never polls the backend.

7. **Draco-compressed glTF for buildings** — Overture tiles are converted to a single Draco-compressed glTF at build time. Target <50MB compressed.

8. **Frustum culling with margin** — Only render elements within the camera's frustum plus a 20% margin.

### 2.2 Constraints

- **Rate limits are hard boundaries.** The backend never exceeds the allowed request rate for any external API.
- **Greater London bounding box is fixed.** lat 51.2–51.7, lon -0.5–0.3.
- **Desktop-first.** Mobile support is out of scope for V1.
- **No persistent storage.** All data is in-memory.
- **Single deployment target.** Static frontend + Express server.

### 2.3 Non-Goals (V1)

- Mobile support
- Performance settings menu
- Keyboard shortcuts
- Accessibility
- LOD for buildings (frustum culling only)
- Multi-city support
- Historical data playback
- Social sharing / export
- WebSocket (SSE is sufficient)

---

## 3. System Architecture

### 3.1 Frontend Architecture

**Responsibilities:**
- Render the 3D scene (buildings, terrain, transport layers)
- Manage camera controls (orbit, zoom, pan, presets)
- Handle user interactions (hover, click, layer toggles, search)
- Display UI overlays (loading screen, tooltips, jamcam, layer toggles)
- Receive vehicle snapshots via SSE
- Manage UI state (selected vehicle, hovered vehicle, visible layers, search query)

**Components:**

```
App (root)
├── LoadingScreen (full-screen, dismisses on ready)
├── SceneProvider (R3F Canvas + PostProcessing)
│   ├── OrbitControls (damping, auto-rotate)
│   ├── AmbientLight + DirectionalLight
│   ├── UnrealBloomPass
│   ├── useGLTF (Draco buildings)
│   ├── GroundPlane (Y=0)
│   ├── ThamesWater (Y=-0.5, animated)
│   ├── Roads (LineSegments)
│   ├── TubeLines (13 colored neon lines + animated dots)
│   ├── BusMarkers (InstancedMesh capsules)
│   ├── TrainMarkers (InstancedMesh spheres)
│   ├── PlaneMarkers (InstancedMesh capsules)
│   ├── ShipMarkers (InstancedMesh capsules)
│   ├── RiverBoats (InstancedMesh spheres)
│   └── StreetLamps (InstancedMesh small boxes)
├── LayerToggles (top-right button group)
├── SearchPanel (search bar + filter results)
├── Tooltip (appears on vehicle hover)
├── JamcamOverlay (appears on signal hover)
└── AttributionFooter (bottom of screen)
```

**State Management (Zustand) — UI state only:**

```typescript
// Zustand store holds ONLY UI state, NOT vehicle positions
interface AppState {
  // UI state
  hoveredVehicle: VehicleSnapshot | null;
  selectedVehicle: VehicleSnapshot | null;
  hoveredSignal: Signal | null;
  visibleLayers: Set<VehicleType>;
  searchQuery: string;
  searchResults: VehicleSnapshot[];
  cameraPreset: 'default' | 'top-down' | 'close-up';
  loading: boolean;
  error: string | null;
  layerStatus: Record<VehicleType, 'ok' | 'stale' | 'error'>;

  // UI actions only
  setHoveredVehicle: (v: VehicleSnapshot | null) => void;
  setSelectedVehicle: (v: VehicleSnapshot | null) => void;
  toggleLayer: (type: VehicleType) => void;
  setSearchQuery: (q: string) => void;
  setCameraPreset: (p: AppState['cameraPreset']) => void;
}
```

**Simulation State (mutable refs — NOT in Zustand):**

```typescript
// In Scene.tsx — accessible via useFrame, NOT React state
const vehiclesRef = useRef<VehicleSnapshot[]>([]);
const buildingsRef = useRef<GLTF | null>(null);
```

**Rendering Architecture:**
- `@react-three/fiber` Canvas is the root of all 3D rendering
- `useFrame` reads from mutable refs directly — no React re-render triggered
- SSE listener updates the refs — no React re-render triggered
- Only UI state changes (hover, click, search) trigger React re-renders

### 3.2 Backend Architecture

**Responsibilities:**
- Poll external APIs on its own schedule (not per-client)
- Normalize responses into a unified transport schema
- Track vehicle identity across polling cycles (entity resolution)
- Run inference engine (bus/train position calculation)
- Maintain canonical vehicle state
- Broadcast vehicle snapshots to all connected clients via SSE
- Serve static map data (Draco glTF + geometry GeoJSON)
- Enforce rate limits per data source
- Handle errors gracefully (circuit breakers, retries)

**Aggregator Design:**

```
┌────────────────────────────────────────────────────────────┐
│                   Backend Aggregator                        │
│                                                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  TfL     │  │ Darwin   │  │  ADSBx   │  │  AIS     │  │
│  │ Adapter  │  │ Adapter  │  │ Adapter  │  │ Adapter  │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
│       │              │              │              │       │
│       └──────────────┴──────────────┴──────────────┘       │
│                            │                                │
│                   ┌────────▼────────┐                       │
│                   │  Normalizer     │                       │
│                   │  (unified       │                       │
│                   │   transport     │                       │
│                   │   schema)       │                       │
│                   └────────┬────────┘                       │
│                            │                                │
│                   ┌────────▼────────┐                       │
│                   │  Vehicle State  │                       │
│                   │  Engine         │                       │
│                   │  - entity       │                       │
│                   │    resolution   │                       │
│                   │  - inference    │                       │
│                   │    engine       │                       │
│                   │  - heartbeat    │                       │
│                   │    monitor      │                       │
│                   └────────┬────────┘                       │
│                            │                                │
│                   ┌────────▼────────┐                       │
│                   │  SSE Broadcast  │                       │
│                   │  Channel        │                       │
│                   └─────────────────┘                       │
└────────────────────────────────────────────────────────────┘
```

**Key difference from v1:** The backend is NOT a proxy. It does NOT forward client requests to external APIs. Instead:
1. Backend polls APIs on its own schedule (one poll = all clients)
2. Backend maintains the canonical vehicle state
3. Backend broadcasts snapshots via SSE to all connected clients
4. If 1 user or 1000 users connect, the backend still only polls once

### 3.3 Data Layer

**Static Data:**
- Buildings: Draco-compressed glTF in `public/map-data/buildings.glb` (~50MB compressed)
- Route geometry: GeoJSON in `public/map-data/geometry.geojson` (~50MB)
- Loaded once at app startup

**Real-Time Data:**
- Transport data: Broadcast via SSE from backend
- Jamcams: Broadcast via SSE from backend
- All data flows through the Vehicle State Engine

**Transformation Pipeline:**

```
External API Response
  → Adapter (auth, HTTP request, error handling)
  → Normalizer (map to unified transport schema)
  → Validator (lat/lng bounds, required fields, type checks)
  → Entity Resolver (match to existing vehicle or create new)
  → Inference Engine (calculate position for buses/trains)
  → Heartbeat Monitor (flag stale entities)
  → SSE Broadcast (push snapshot to all clients)
```

---

## 4. Component Architecture

### 4.1 Rendering Engine

| Property | Value |
|----------|-------|
| **Responsibility** | Render the 3D scene using Three.js via React Three Fiber |
| **Inputs** | Mutable refs (vehicle positions), camera position |
| **Outputs** | Rendered WebGL canvas |
| **Dependencies** | Three.js, @react-three/fiber, @react-three/drei, @react-three/postprocessing, draco3d |
| **Failure behavior** | If Three.js fails, show error overlay with message |

### 4.2 Vehicle State Engine (NEW — critical addition)

| Property | Value |
|----------|-------|
| **Responsibility** | Maintain canonical vehicle state, track identity across polling cycles, run inference, broadcast snapshots |
| **Inputs** | Normalized vehicle data from adapters |
| **Outputs** | VehicleSnapshot[] broadcast via SSE |
| **Dependencies** | Route geometry, inference engine, entity resolver |
| **Failure behavior** | If engine crashes, SSE stops, frontend shows "data unavailable" |

**Sub-components:**

**Entity Resolver:**
- Matches incoming vehicles to existing tracked vehicles using stable identifiers
- For GPS data (planes, ships, GPS buses): match by `icao24` / `mmsi` / `registration`
- For inferred data (trains, inferred buses): match by `route + direction + scheduled_time_window`
- When a match is found, update position. When no match, create new entry. When a vehicle disappears for >2 polling cycles, mark as stale, then remove.

**Inference Engine:**
- Calculates position along route geometry for buses and trains
- Receives: route geometry, ETA, current time
- Outputs: { x, y, z, heading } in Three.js world coordinates
- Runs on the backend (not frontend)

**Heartbeat Monitor:**
- Flags vehicles that haven't moved despite ETA changes
- Logs warnings for stale entities
- Helps debug inference accuracy

### 4.3 Transport Layers (Frontend)

Each transport layer follows the same pattern:

| Property | Value |
|----------|-------|
| **Responsibility** | Render vehicles of one type, update positions from refs each frame |
| **Inputs** | Mutable ref (vehicle snapshots), route geometry |
| **Outputs** | Three.js mesh instances |
| **Dependencies** | Refs, geo utilities |
| **Failure behavior** | Render empty layer (no vehicles visible) |

### 4.4 Map Data Pipeline

| Property | Value |
|----------|-------|
| **Responsibility** | Download Overture tiles, convert to Draco-compressed glTF, serve to frontend |
| **Inputs** | Overture Parquet tiles (downloaded), Overpass API response |
| **Outputs** | Draco-compressed glTF + GeoJSON served as static files |
| **Dependencies** | draco3d, parquetjs (Node.js), osmtogeojson (Node.js) |
| **Failure behavior** | If tiles missing, render empty scene (no buildings) |

**Pipeline (changed from v1):**
```
Overture S3 → Download Parquet tiles → Convert to merged glTF with Draco compression → Save to public/map-data/buildings.glb
Overpass API → Fetch geometry → Convert to GeoJSON → Save to public/map-data/geometry.geojson
Frontend → Load glTF via useGLTF → Three.js renders buildings
```

**Target: <50MB compressed glTF** (vs. 400-800MB GeoJSON in v1)

### 4.5 API Adapters

| Property | Value |
|----------|-------|
| **Responsibility** | Fetch data from external APIs, normalize to unified schema |
| **Inputs** | None (stateless, called with parameters) |
| **Outputs** | Normalized data in unified transport schema |
| **Dependencies** | Node.js fetch, process.env for credentials |
| **Failure behavior** | Return empty array, log error, state engine marks layer as 'error' |

**Unified Transport Schema (NEW — critical addition):**

All adapters output the same shape. The frontend never sees raw API responses.

```typescript
// Unified schema — consumed by both backend and frontend
interface TransportEntity {
  // Identity
  entityId: string;       // Stable ID: icao24, mmsi, registration, or route+time
  type: VehicleType;
  route: string;
  registration?: string;

  // Position (Three.js world coordinates, NOT lat/lng)
  x: number;              // Easting
  y: number;              // Altitude/elevation
  z: number;              // Northing
  heading: number;        // Degrees

  // Movement
  speed?: number;
  vertRate?: number;

  // Metadata
  destination: string;
  nextStops?: string[];
  eta?: number;
  delay?: number;
  status?: 'on_time' | 'delayed' | 'cancelled';
  dataCertainty: 'gps' | 'inferred';  // NEW: visual distinction

  // Source
  sourceType: 'tfl' | 'darwin' | 'adsbx' | 'ais';
  timestamp: number;      // Unix epoch
}
```

**Per-adapter specifics:**

| Adapter | Endpoint | Auth | Normalization |
|---------|----------|------|---------------|
| TfL | Multiple | API key header | Map lineName, destination, minutesToStation → TransportEntity |
| Darwin | /departure-board/{code} | API key header | Map TargetArrival, DestinationName → TransportEntity |
| ADSBx | /area?lat=&lon=&rad= | x-api-key header | Map callsign, lat, lng, altitude, heading → TransportEntity |
| AIS Hub | /ws.php?username=... | Username query param | Map NAME, MMSI, LATITUDE, LONGITUDE, SOG, HEADING → TransportEntity |

### 4.6 SSE Broadcast Layer

| Property | Value |
|----------|-------|
| **Responsibility** | Push vehicle snapshots to all connected clients |
| **Inputs** | VehicleStateEngine output |
| **Outputs** | SSE events to frontend |
| **Dependencies** | Node.js SSE, VehicleStateEngine |
| **Failure behavior** | If SSE fails, client reconnects automatically |

**SSE Event Format:**

```typescript
// Event: "vehicles"
// Data: {
//   type: "vehicles",
//   snapshot: TransportEntity[]
// }

// Event: "jamcams"
// Data: {
//   type: "jamcams",
//   snapshot: Jamcam[]
// }

// Event: "layer-status"
// Data: {
//   type: "layer-status",
//   status: Record<VehicleType, 'ok' | 'stale' | 'error'>
// }
```

**Polling Schedule (backend-driven):**

| Data Type | Poll Interval | SSE Broadcast |
|-----------|--------------|---------------|
| Tube | 30s | Every poll |
| Bus | 30s | Every poll |
| Train | 30s | Every poll |
| Plane | 10s | Every poll |
| Ship | 60s | Every poll |
| River Boat | 30s | Every poll |
| Jamcam | 60s | Every poll |
| Layer Status | 30s | Every poll |

### 4.7 UI Layer

| Property | Value |
|----------|-------|
| **Responsibility** | Render HTML overlays, handle user interactions |
| **Inputs** | Zustand store (UI state only), mutable refs (for rendering) |
| **Outputs** | HTML elements overlaid on the 3D canvas |
| **Dependencies** | Zustand store, component libraries |
| **Failure behavior** | UI degrades gracefully (no tooltips, no overlays) |

---

## 5. Data Architecture

### 5.1 Domain Entities

**TransportEntity** (unified schema, replaces per-type Vehicle):
```
TransportEntity
├── identity
│   ├── entityId: string      // Stable ID (icao24, mmsi, registration, or route+time)
│   ├── type: VehicleType     // 'tube' | 'bus' | 'train' | 'plane' | 'ship' | 'riverBoat'
│   ├── route: string         // Route number/name
│   └── registration?: string // Vehicle registration
├── position (Three.js world coords)
│   ├── x: number             // Easting (Web Mercator)
│   ├── y: number             // Altitude/elevation
│   ├── z: number             // Northing (Web Mercator)
│   └── heading: number       // Degrees
├── movement
│   ├── speed?: number        // m/s (planes) or knots (ships)
│   └── vertRate?: number     // m/min (planes only)
├── metadata
│   ├── destination: string
│   ├── nextStops?: string[]
│   ├── eta?: number          // Seconds to next stop
│   ├── delay?: number        // Seconds of delay
│   ├── status?: string
│   └── dataCertainty: 'gps' | 'inferred'  // NEW: visual distinction
└── source
    ├── sourceType: string    // 'tfl' | 'darwin' | 'adsbx' | 'ais'
    └── timestamp: number     // Unix epoch
```

**Jamcam:**
```
Jamcam
├── id: string
├── name: string
├── x: number             // Three.js world coords (converted from lat/lng)
├── y: number
├── z: number
├── available: boolean
├── imageUrl: string
├── videoUrl: string
└── view: string
```

**Signal (Street Lamp):**
```
Signal
├── id: string
├── x: number
├── y: number
├── z: number
├── jamcamId?: string
└── roadSegment: string
```

**RouteGeometry:**
```
RouteGeometry
├── id: string
├── type: GeometryType
├── name: string
└── points: { x: number; y: number; z: number }[]  // Already in Three.js world coords
```

### 5.2 Data Ownership

| Data | Owned By | Updated By |
|------|----------|------------|
| Transport entities | Backend VehicleStateEngine | Backend adapters (polling) |
| Map data | Frontend (static files) | Build scripts (pre-processed) |
| UI state | Frontend Zustand | User interaction |

### 5.3 Data Lifecycle

```
1. Server boot → Load static map data (glTF + geometry)
2. Server boot → Start polling timers per data type
3. Poll timer fires → Fetch external API → Normalize → Entity Resolve → Inference → Heartbeat
4. VehicleStateEngine → SSE Broadcast → All clients
5. Client SSE listener → Update mutable refs → useFrame reads refs → GPU renders
6. React only re-renders for UI state changes (hover, click, search, layer toggle)
```

### 5.4 Update Frequency

Same as v1. Backend polls on its schedule, broadcasts to all clients.

---

## 6. Runtime Data Flow

### 6.1 Tube Data Flow

```
TfL API /Metro/Live/Arrivals/{stationId}
  → TfL Adapter (auth: API key)
  → Normalizer (map to unified TransportEntity)
  → Validator (lat/lng bounds, required fields)
  → Entity Resolver (match to existing tube entity or create new)
  → Inference Engine (calculate x,y,z on line geometry from ETA)
  → Heartbeat Monitor (check for stale entities)
  → VehicleStateEngine (update canonical state)
  → SSE Broadcast (push snapshot to all clients)
  → Client SSE listener → Update mutable refs
  → useFrame reads refs → GPU renders neon line + moving dots
```

### 6.2 Bus Data Flow

```
TfL API /Bus/RealTime + /Metro/Live/Arrivals/{stationId}
  → TfL Adapter (auth: API key)
  → Normalizer (GPS buses: direct position; inferred: mark as 'inferred')
  → Validator (check lat/lng, snap to road geometry)
  → Entity Resolver (match by registration for GPS, route+time for inferred)
  → Inference Engine (calculate x,y,z along road geometry)
  → Heartbeat Monitor
  → VehicleStateEngine
  → SSE Broadcast
  → Client SSE → mutable refs → useFrame → GPU renders red capsules
```

### 6.3 Train Data Flow

```
Darwin DBRS /departure-board/{NaptocCode}
  → Darwin Adapter (auth: API key)
  → Normalizer (map to unified TransportEntity)
  → Validator (check lat/lng, snap to rail geometry)
  → Entity Resolver (match by route + direction + scheduled_time_window)
  → Inference Engine (calculate x,y,z along rail geometry)
  → Heartbeat Monitor
  → VehicleStateEngine
  → SSE Broadcast
  → Client SSE → mutable refs → useFrame → GPU renders yellow dots
```

### 6.4 Plane Data Flow

```
ADSBx v2 /area?lat=51.5&lon=-0.1&rad=25
  → ADSBx Adapter (auth: x-api-key, Accept-Encoding: gzip)
  → Normalizer (map to unified TransportEntity, dataCertainty: 'gps')
  → Validator (check lat/lng, altitude > 0, is_on_ground === false)
  → Entity Resolver (match by icao24)
  → Inference Engine (not needed — GPS data, direct position)
  → Heartbeat Monitor
  → VehicleStateEngine
  → SSE Broadcast
  → Client SSE → mutable refs → useFrame → GPU renders white capsules
```

### 6.5 Ship Data Flow

```
AIS Hub /ws.php?username=...&latmin=...&latmax=...&lonmin=...&lonmax=...
  → AIS Adapter (auth: username query param)
  → Normalizer (map to unified TransportEntity, dataCertainty: 'gps')
  → Validator (check lat/lng, SOG >= 0, HEADING 0-360)
  → Entity Resolver (match by MMSI)
  → Inference Engine (not needed — GPS data, direct position)
  → Heartbeat Monitor
  → VehicleStateEngine
  → SSE Broadcast
  → Client SSE → mutable refs → useFrame → GPU renders teal capsules
```

### 6.6 Jamcam Data Flow

```
TfL API /Place/Type/JamCam
  → TfL Adapter (auth: API key)
  → Normalizer (map to Jamcam[], convert lat/lng to Three.js world coords)
  → Validator (check lat/lng, available === true)
  → SSE Broadcast
  → Client SSE → mutable refs → StreetLamps component assigns nearest jamcam
  → Hover on signal → JamcamOverlay shows imageUrl/videoUrl
```

---

## 7. Rendering Architecture

### 7.1 Three.js Scene Structure

```
Scene
├── Camera: PerspectiveCamera (fov: 60, near: 1, far: 50000)
├── Lights:
│   ├── AmbientLight (0x404060, intensity: 0.3)
│   └── DirectionalLight (0xffeedd, intensity: 0.5, castShadow: true)
├── Post-processing:
│   └── EffectComposer + UnrealBloomPass
├── Geometry (static, loaded once):
│   ├── Buildings (Draco-compressed glTF, loaded via useGLTF)
│   ├── GroundPlane (PlaneGeometry, Y=0)
│   ├── ThamesWater (PlaneGeometry, Y=-0.5, animated)
│   ├── Roads (LineSegments from OSM geometry)
│   ├── TubeLines (13 LineSegments, one per line)
│   ├── RailLines (LineSegments from OSM geometry)
│   └── Waterway (LineSegments for Thames)
├── Geometry (dynamic, updated each frame from refs):
│   ├── StreetLamps (InstancedMesh, ~500 instances)
│   ├── TubeDots (InstancedMesh, ~200 instances)
│   ├── BusMarkers (InstancedMesh, ~800 instances)
│   ├── TrainDots (InstancedMesh, ~100 instances)
│   ├── PlaneMarkers (InstancedMesh, ~50 instances)
│   ├── ShipMarkers (InstancedMesh, ~20 instances)
│   └── RiverBoatDots (InstancedMesh, ~10 instances)
```

### 7.2 R3F Component Hierarchy

```tsx
<Canvas gl={{ antialias: true, alpha: false, powerPreference: 'high-performance' }}
        dpr={[1, 2]}
        camera={{ position: [0, 2000, 2000], fov: 60 }}>
  <OrbitControls enableDamping dampingFactor={0.05} autoRotate={false} />
  <ambientLight intensity={0.3} color={0x404060} />
  <directionalLight intensity={0.5} color={0xffeedd} castShadow />
  <EffectComposer>
    <UnrealBloomPass resolution={256} strength={1.5} radius={0.4} threshold={0.6} />
  </EffectComposer>
  <MapFoundation />
  <TransportLayers />
</Canvas>
```

### 7.3 Instancing Strategy

**Buildings:**
- Draco-compressed glTF loaded via `useGLTF` from Drei
- Single file, ~50MB compressed
- Three.js handles LOD and frustum culling internally
- No manual instancing needed for buildings

**Vehicles:**
- One `InstancedMesh` per transport type
- ~6 instanced meshes total for all vehicles
- Each instance has its own transform matrix (position + rotation) and color

**Total draw calls:** ~210 (buildings ~10-20 via glTF + 6 vehicles + ~200 lines/roads/lamps)

### 7.4 LOD Strategy

**V1:** Frustum culling only. Three.js handles this automatically for glTF models. No manual LOD.

**Future enhancement:** If VRAM becomes an issue, implement distance-based building simplification.

### 7.5 Coordinate System

| Axis | Meaning | Range |
|------|---------|-------|
| X | Easting (Web Mercator) | -5000 to +5000 meters |
| Y | Altitude/Elevation | -0.5 (river) to +5000 (air) |
| Z | Northing (Web Mercator) | -5000 to +5000 meters |

**Origin:** London center (51.5074°N, -0.1278°W) → (0, 0, 0)

**Conversion:** Web Mercator projection (EPSG:3857) converts lat/lng to X/Z in meters.

### 7.6 Animation Loop

All animation happens in R3F's `useFrame` hook. **No React re-render is triggered:**

```typescript
// In Scene.tsx — reads from mutable refs, NOT Zustand
useFrame(() => {
  const delta = clock.getDelta();

  // Read vehicle positions from refs (no React re-render)
  const vehicles = vehiclesRef.current;

  // Update instance matrices directly
  vehicles.forEach((vehicle, i) => {
    const mesh = vehicleMeshesRef.current[vehicle.type];
    if (!mesh) return;

    const matrix = new THREE.Matrix4();
    matrix.setPosition(vehicle.x, vehicle.y, vehicle.z);
    matrix.makeRotationY(vehicle.heading * Math.PI / 180);
    mesh.setMatrixAt(i, matrix);
  });

  // Mark instances for update
  vehiclesRef.current.forEach(v => {
    const mesh = vehicleMeshesRef.current[v.type];
    if (mesh) mesh.instanceMatrix.needsUpdate = true;
  });
});
```

**SSE listener updates refs (no React re-render):**

```typescript
// In Scene.tsx — SSE listener
useEffect(() => {
  const eventSource = new EventSource('/api/sse/transport');
  eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    if (data.type === 'vehicles') {
      // Update mutable refs directly — NO React re-render
      vehiclesRef.current = data.snapshot;
    }
  };
  return () => eventSource.close();
}, []);
```

---

## 8. Backend Architecture

### 8.1 Routes

```typescript
// server/routes.ts
const router = express.Router();

// SSE streams (long-lived connections)
router.get('/api/sse/transport', sseController.transportStream);
router.get('/api/sse/jamcams', sseController.jamcamStream);
router.get('/api/sse/layer-status', sseController.layerStatusStream);

// Static map data
router.get('/api/map/buildings', (req, res) => res.sendFile('buildings.glb'));
router.get('/api/map/geometry', (req, res) => res.sendFile('geometry.geojson'));

// Search
router.get('/api/search', searchController.search);

// Health check
router.get('/api/health', (req, res) => res.json({ status: 'ok' }));
```

### 8.2 Services

| Service | File | Responsibility |
|---------|------|----------------|
| Adapters | `server/adapters/tfl.ts`, `darwin.ts`, `adsbx.ts`, `ais.ts` | Fetch + normalize from external APIs |
| Normalizer | `server/normalizer/transport.ts` | Map to unified TransportEntity schema |
| EntityResolver | `server/tracking/entityResolver.ts` | Match incoming vehicles to existing entities |
| InferenceEngine | `server/inference/bus.ts`, `train.ts` | Calculate position along geometry |
| HeartbeatMonitor | `server/tracking/heartbeat.ts` | Flag stale entities |
| VehicleStateEngine | `server/tracking/vehicleStateEngine.ts` | Maintain canonical state, coordinate all components |
| SSEController | `server/sse/controller.ts` | Manage SSE connections, broadcast snapshots |
| MapController | `server/controllers/map.ts` | Serve static map files |
| SearchController | `server/controllers/search.ts` | Search/filter vehicles |

### 8.3 Cache / State Strategy

The backend does NOT use a traditional cache. Instead, it uses a **Vehicle State Engine** that maintains the canonical state:

```typescript
// server/tracking/vehicleStateEngine.ts
class VehicleStateEngine {
  // Map<entityId, TransportEntity>
  private vehicles = new Map<string, TransportEntity>();

  // Polling schedule
  private pollingSchedule: Record<VehicleType, number> = {
    tube: 30000,
    bus: 30000,
    train: 30000,
    plane: 10000,
    ship: 60000,
    riverBoat: 30000,
  };

  // Heartbeat timeout (seconds before marking stale)
  private heartbeatTimeout = 90;

  // SSE subscribers
  private subscribers: Set<Response> = new Set();

  // Poll all sources on their schedule
  start(): void {
    Object.entries(this.pollingSchedule).forEach(([type, interval]) => {
      setInterval(() => this.poll(type), interval);
    });
  }

  // Poll a specific transport type
  async poll(type: VehicleType): Promise<void> {
    try {
      const rawData = await this.fetch(type);       // Adapter
      const normalized = this.normalize(rawData);    // Normalizer
      const resolved = this.resolve(normalized);     // Entity Resolver
      const inferred = this.infer(resolved);         // Inference Engine
      this.heartbeat(inferred);                      // Heartbeat Monitor
      this.broadcast(inferred);                      // SSE Broadcast
    } catch (err) {
      this.handlePollError(type, err);               // Circuit breaker
    }
  }

  // Register SSE subscriber
  subscribe(res: Response): void {
    this.subscribers.add(res);
    res.on('close', () => this.subscribers.delete(res));
  }

  // Broadcast to all SSE subscribers
  private broadcast(entities: TransportEntity[]): void {
    const payload = JSON.stringify({ type: 'vehicles', snapshot: entities });
    this.subscribers.forEach(sub => sub.write(`data: ${payload}\n\n`));
  }
}
```

### 8.4 Error Handling

| Error Type | Handling |
|------------|----------|
| Network timeout | Retry once after 2s, then mark layer as 'error' in layer status |
| HTTP 429 (rate limit) | Wait for rate limit window, retry once |
| HTTP 5xx | Retry once after 1s, then mark layer as 'error' |
| Invalid response | Log error, skip that vehicle, continue with others |
| Missing required fields | Log warning, skip that vehicle |
| Coordinates out of bounds | Log warning, skip that vehicle |

**Circuit Breaker:** If an adapter fails 3 consecutive polls, mark the layer as 'error' and stop polling for 60s. After 60s, retry once.

### 8.5 Retries

- **Max retries:** 1 per request
- **Retry delay:** 1s for 5xx, 2s for network timeout
- **No exponential backoff**

### 8.6 Rate Limiting

Token bucket algorithm per data source:

```typescript
class RateLimiter {
  private buckets = new Map<string, { tokens: number; lastRefill: number }>();

  constructor(private limits: Record<string, number>) {
    // limits: { adsbx: 0.1, ais: 0.0167, tfl: 10, darwin: 10 } (requests per second)
    for (const [source, rate] of Object.entries(limits)) {
      this.buckets.set(source, { tokens: rate, lastRefill: Date.now() });
    }
  }

  async acquire(source: string): Promise<void> {
    const bucket = this.buckets.get(source);
    if (!bucket) return;

    const now = Date.now();
    const elapsed = (now - bucket.lastRefill) / 1000;
    bucket.tokens = Math.min(this.limits[source], bucket.tokens + elapsed * this.limits[source]);
    bucket.lastRefill = now;

    if (bucket.tokens < 1) {
      const wait = (1 - bucket.tokens) / this.limits[source];
      await new Promise(r => setTimeout(r, wait * 1000));
      bucket.lastRefill = Date.now();
      bucket.tokens = 0;
    } else {
      bucket.tokens -= 1;
    }
  }
}
```

---

## 9. Deployment Architecture

### 9.1 Development Environment

```
Terminal session:
  npm run dev
    ├── vite (port 5173) — Frontend dev server with HMR
    └── node server/index.ts (port 3001) — Backend aggregator

Vite config proxy:
  server.proxy['/api/sse'] = 'http://localhost:3001'
  server.proxy['/api/map'] = 'http://localhost:3001'

Environment:
  .env.local — API keys (gitignored)
  .env.example — Template (committed)
```

### 9.2 Production Deployment

**Recommended: Single server (Option B)**

- Express server serves both static files (from `dist/`) and API routes
- Deployed via Docker or direct to VPS
- Simpler than separate frontend/backend hosting
- Fewer moving parts

**Deployment steps:**
1. Run build scripts: `npm run build:map-data` (download + convert Overture tiles)
2. Build frontend: `npm run build` (Vite produces `dist/`)
3. Start server: `node server/index.ts` (serves `dist/` + API routes)

**Alternative: Separate hosting**
- Frontend: Vercel/Netlify (static files)
- Backend: Railway/Render (Express server)
- Map data: S3/CDN (glTF + GeoJSON files)

### 9.3 Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `TFL_API_KEY` | Yes | TfL API key |
| `DARWIN_API_KEY` | Yes | Darwin DBRS API key |
| `ADSBX_API_KEY` | Yes | ADSBx API key |
| `AIS_HUB_USERNAME` | Yes | AIS Hub username |
| `PORT` | No (default: 3001) | Server port |

### 9.4 Static Assets

| Asset | Location | Generated By |
|-------|----------|--------------|
| Buildings glTF | `public/map-data/buildings.glb` | `scripts/download-and-convert-buildings.ts` |
| Geometry GeoJSON | `public/map-data/geometry.geojson` | `scripts/fetch-osm-geometry.ts` |
| Built frontend | `dist/` | `vite build` |

---

## 10. Security Architecture

### 10.1 API Key Handling

| Layer | Handling |
|-------|----------|
| Storage | `.env.local` (gitignored), read via `process.env` |
| Transmission | HTTPS only for all external API calls |
| Frontend | Never exposed — all API calls go through backend |
| Logging | API keys are never logged or printed |

### 10.2 Secrets Management

- **Development:** `.env.local` file
- **Production:** Environment variables set by hosting platform
- **No secret managers**

### 10.3 Input Validation

All external API responses validated before entering the Vehicle State Engine:

```typescript
function validateEntity(data: unknown, type: VehicleType): TransportEntity | null {
  if (!data || typeof data !== 'object') return null;
  const obj = data as Record<string, unknown>;

  // Required fields
  if (typeof obj.lat !== 'number' || typeof obj.lng !== 'number') return null;

  // Bounds check (Greater London + buffer)
  if (obj.lat < 51.0 || obj.lat > 52.0 || obj.lng < -1.0 || obj.lng > 1.0) return null;

  // Type-specific validation
  if (type === 'plane' && (typeof obj.altitude !== 'number' || obj.altitude < 0)) return null;
  if (type === 'ship' && (typeof obj.speed !== 'number' || obj.speed < 0)) return null;

  return mapToEntity(obj, type);
}
```

### 10.4 External API Failures

| Scenario | Behavior |
|----------|----------|
| API returns 5xx | Retry once, mark layer as 'error' |
| API times out | Retry once, mark layer as 'error' |
| API returns malformed JSON | Log error, skip that vehicle |
| API key invalid | Log error, mark layer as 'error', stop polling for 60s |
| Rate limit exceeded | Wait and retry, log warning |

---

## 11. Observability

### 11.1 Logging

**Backend:**
- `[TFL] Fetched 247 stations in 340ms`
- `[ADSBX] Fetched 42 aircraft in 1200ms`
- `[EntityResolver] Matched 38/42 planes, created 4 new, removed 2 stale`
- `[Inference] Bus N343: position 0.73 along route (ETA 95s)`
- `[Heartbeat] WARNING: Train 9100GLVC_001 stale for 120s`
- `[CircuitBreaker] TFL adapter marked as error after 3 failures`

**Frontend:**
- `[SSE] Connected, received 1042 vehicles`
- `[SSE] Disconnected, reconnecting in 5s`
- `[Layer] Bus layer status: stale (last update 45s ago)`

### 11.2 Metrics

**V1:** No external monitoring. Performance verified visually.

**Future:** API response times, cache hit rate, vehicle count, frame rate.

### 11.3 Debugging Approach

1. **Visual debugging:** Toggle layers, zoom, check positions
2. **Console logging:** Backend logs show fetch times, entity resolution, inference
3. **SSE inspection:** Browser DevTools Network tab shows SSE events
4. **Store inspection:** `useStore.getState()` in browser console
5. **Ref inspection:** `vehiclesRef.current` in browser console (from Scene component)

---

## 12. Architecture Decision Records

### ADR-001: Dummy-First Development
- **Status:** Accepted
- **Context:** API keys required for all sources. Rate limits constrain testing.
- **Decision:** Build and test entire frontend with dummy data. Real API integration is a drop-in replacement.
- **Consequences:** Faster iteration. Visual feedback immediately.

### ADR-002: Instanced Meshes for Vehicles
- **Status:** Accepted
- **Context:** Hundreds of vehicles, need minimal draw calls.
- **Decision:** One `InstancedMesh` per transport type.
- **Consequences:** ~6 draw calls for all vehicles. Per-instance colors via `setColorAt()`.

### ADR-003: Geometry-Constrained Animation
- **Status:** Accepted
- **Context:** Vehicles must appear on roads/tracks.
- **Decision:** All positions projected onto nearest point of geometry array.
- **Consequences:** Always correct visual placement. Requires OSM geometry.

### ADR-004: Draco-Compressed glTF for Buildings (REVISED from v1)
- **Status:** Accepted
- **Context:** Static GeoJSON (400-800MB) will crash the browser. JSON.parse freezes main thread for 10-30s. Heap expansion 2-4x raw size.
- **Decision:** Download Overture Parquet tiles, convert to single merged glTF with Draco compression at build time. Target <50MB compressed. Load via `useGLTF` from Drei.
- **Consequences:** Zero JSON.parse overhead. Loads in ~2-3s. Three.js handles LOD/frustum internally. Requires build script with draco3d. Build script must be run locally (not in CI/CD) due to memory requirements.
- **Build script location:** `scripts/download-and-convert-buildings.ts`
- **Output:** `public/map-data/buildings.glb`

### ADR-005: Backend Aggregator, Not Proxy (REVISED from v1)
- **Status:** Accepted
- **Context:** Proxy model fails with 2+ users — each client triggers separate API calls, exhausting rate limits.
- **Decision:** Backend polls APIs on its own schedule. Maintains canonical vehicle state. Broadcasts to all clients via SSE. One poll serves all users.
- **Consequences:** Scales to unlimited clients without additional API calls. Adds complexity (SSE, state management, entity resolution). Backend must hold map geometry for inference.

### ADR-006: SSE Over HTTP Polling (REVISED from v1)
- **Status:** Accepted
- **Context:** HTTP polling creates unnecessary requests. SSE is push-based, lighter.
- **Decision:** Backend pushes vehicle snapshots via SSE. Frontend uses `EventSource`.
- **Consequences:** Real-time updates. Automatic reconnection. Simpler frontend (no polling loop). Limited browser support (SSE supported in all modern browsers).

### ADR-007: Mutable Refs for Vehicle Positions (NEW)
- **Status:** Accepted
- **Context:** Updating 1000+ vehicles through Zustand every 10-30s triggers React re-renders of all components. Even with `useShallow`, the reconciliation cycle causes frame drops.
- **Decision:** Vehicle positions stored in mutable refs. `useFrame` reads refs directly. Only UI state (hover, click, search) stored in Zustand.
- **Consequences:** No React re-render on position updates. 60fps maintained. Slightly less "React-idiomatic" but necessary for performance.

### ADR-008: Unified Transport Schema (NEW)
- **Status:** Accepted
- **Context:** 5 external APIs with different response formats. Brittle — any field change breaks the frontend.
- **Decision:** All adapters output the same `TransportEntity` shape. Frontend consumes only this schema.
- **Consequences:** Frontend isolated from API changes. Adapter pattern makes provider swapping easy. Adds a normalization layer.

### ADR-009: Entity Resolution Layer (NEW)
- **Status:** Accepted
- **Context:** Without stable identifiers, vehicles despawn/respawn every polling cycle. A train at Station A then Station B must be recognized as the same entity.
- **Decision:** Backend Entity Resolver matches incoming vehicles to existing entities using stable IDs (icao24, mmsi, registration) or route+direction+time windows.
- **Consequences:** Smooth vehicle transitions. No flickering. Adds complexity to the backend.

### ADR-010: Inference Engine on Backend (REVISED from v1)
- **Status:** Accepted
- **Context:** Running inference on frontend wastes CPU across all clients. Backend has the geometry anyway.
- **Decision:** Inference runs on backend. Frontend receives render-ready snapshots (x,y,z coordinates).
- **Consequences:** Consistent data across all clients. Backend holds map geometry. Frontend is simpler (just renders).

### ADR-011: Polling Over WebSockets
- **Status:** Accepted
- **Context:** External APIs are HTTP-based. No WebSocket endpoints.
- **Decision:** HTTP polling (backend) + SSE (backend to frontend).
- **Consequences:** Simpler than WebSockets. Acceptable latency.

### ADR-012: Visual Distinction — GPS vs Inferred (NEW)
- **Status:** Accepted
- **Context:** Users may mistake inferred positions for precise GPS coordinates.
- **Decision:** `dataCertainty: 'gps' | 'inferred'` field. GPS vehicles: solid glow. Inferred vehicles: dashed trail or slightly dimmer glow.
- **Consequences:** Transparency about data quality. Slightly more complex rendering.

### ADR-013: Single Server Deployment (REVISED from v1)
- **Status:** Accepted
- **Context:** Separate frontend/backend hosting adds complexity.
- **Decision:** Express server serves static files + API routes. Single deployment target.
- **Consequences:** Simpler ops. Fewer moving parts. Easier debugging.

### ADR-014: Attribution Overlay (NEW)
- **Status:** Accepted
- **Context:** Overture, OSM, TfL have strict attribution requirements.
- **Decision:** Persistent footer showing data source credits.
- **Consequences:** Legal compliance. Minimal UI impact.

---

## 13. Assumptions

1. **Overture tiles exist** for Greater London. Spike tested with 1km² area before full download.
2. **Overpass API returns complete geometry** for major roads, rail lines, waterways. Minor routes may be missing.
3. **TfL API keys obtained** before API integration. Registration is free and instant.
4. **ADSBx free tier provides sufficient coverage** for London airspace.
5. **AIS Hub provides Thames coverage.**
6. **Three.js instanced meshes support per-instance colors** via `setColorAt()`.
7. **Web Mercator projection is accurate enough** for London-scale visualization.
8. **Draco compression reduces building data to <50MB** for Greater London. Spike-tested.
9. **SSE is supported** in all target browsers (Chrome, Firefox, Safari, Edge — all support EventSource).
10. **Darwin DBRS credits are sufficient** for 30s polling of ~200 stations. Estimated ~200 calls/hour × 16 hours = 3,200 credits/day. Well within 100k limit.

---

## 14. Open Questions

1. **What if Draco compression doesn't achieve <50MB?** Fallback: split buildings into 4 zone files, lazy-load by camera position.
2. **What is the minimum supported browser?** Three.js requires WebGL 1.0+. Chrome 56+, Firefox 51+, Safari 11+, Edge 16+.
3. **Should street lamps be on all roads or just major roads?** Currently planned for major roads only.
4. **How to handle OSM geometry validation?** Bus routes may be broken/incomplete. Need validation tooling. Spike-tested with real Overpass queries.
5. **What if TfL arrivals data is insufficient for train placement?** Darwin provides more detail for National Rail. Tube trains are the hardest to place accurately. May need to simplify to station-dot visualization for some lines.

---

## 15. Glossary

| Term | Definition |
|------|------------|
| **ADSBx** | ADS-B Exchange, global aircraft tracking network |
| **AIS** | Automatic Identification System, vessel tracking |
| **Darwin DBRS** | National Rail's Departure Board Reporting System |
| **Entity Resolver** | Backend component that matches incoming vehicles to existing tracked entities |
| **EPSG:3857** | Web Mercator projection |
| **GeoJSON** | JSON-based format for geographic data |
| **glTF** | 3D asset format, supports Draco compression |
| **InstancedMesh** | Three.js class for rendering many objects with one draw call |
| **JAMCAM** | TFL's traffic camera system |
| **LOD** | Level of Detail |
| **NaptocCode** | National Public Transport Access Code |
| **Overture Maps** | Open-source map data project |
| **Overpass API** | Read-only API for querying OpenStreetMap data |
| **Parquet** | Columnar storage format |
| **R3F** | React Three Fiber |
| **SSE** | Server-Sent Events |
| **TfL** | Transport for London |
| **TransportEntity** | Unified schema for all transport data |
| **UnrealBloomPass** | Three.js post-processing glow effect |
| **Vehicle State Engine** | Backend component maintaining canonical vehicle state |
| **Web Mercator** | Map projection used by most web maps |
| **Zustand** | Lightweight React state management library |
