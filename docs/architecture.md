# London Live 3D Map — Architecture Document

> **Status:** Draft  
> **Version:** 1.0  
> **Date:** 2026-06-18  
> **Author:** Lead Architect  
> **Audience:** Engineers implementing the system

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
┌────────────────────────────────────────────────────────────┐
│                    Backend Proxy (Express)                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  TfL     │  │ Darwin   │  │  ADSBx   │  │  AIS     │  │
│  │ Fetcher  │  │ Fetcher  │  │ Fetcher  │  │ Fetcher  │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
│       │              │              │              │       │
│       └──────────────┴──────────────┴──────────────┘       │
│                            │                                │
│                   ┌────────▼────────┐                       │
│                   │  In-Memory      │                       │
│                   │  Cache (TTL)    │                       │
│                   └────────┬────────┘                       │
│                            │                                │
│  GET /api/tube             │                                │
│  GET /api/buses            │                                │
│  GET /api/trains           │                                │
│  GET /api/planes           │                                │
│  GET /api/ships            │                                │
│  GET /api/jamcams          │                                │
│  GET /api/map/buildings    │                                │
│  GET /api/map/geometry     │                                │
└────────────────────────────┼────────────────────────────────┘
                             │ fetch()
                             ▼
┌────────────────────────────────────────────────────────────┐
│                    Frontend (Vite + React)                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  Store   │  │  R3F     │  │  UI      │  │  Loading │  │
│  │  (Zustand)│ │  Scene   │  │  Overlay │  │  Screen  │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
└────────────────────────────────────────────────────────────┘
```

---

## 2. Architecture Principles

### 2.1 Key Principles

1. **Dummy-first development** — All data endpoints serve realistic dummy data during development. Real API integration is a drop-in replacement at the end. This de-risks the entire project by allowing visual iteration without API keys or rate limits.

2. **Geometry-constrained animation** — Vehicles never move through free space. Bus and train positions are always projected onto the nearest point of the road/rail geometry array. This guarantees visual correctness (no flying over buildings).

3. **Instanced rendering** — All vehicles of the same type share a single `InstancedMesh`. This reduces draw calls from O(n) to O(1) per transport type. Buildings are merged by height tier into 4 instanced meshes.

4. **Single source of truth** — The Zustand store is the canonical state. No component holds local vehicle data. Components subscribe to store slices via `useStore(selector)`.

5. **Polling over WebSockets** — All external data uses HTTP polling with configurable intervals. No WebSocket connections, no Server-Sent Events. This matches the APIs' design and keeps the architecture simple.

6. **Frustum culling with margin** — Only render elements within the camera's frustum plus a 20% margin. This creates the illusion of instant loading while keeping GPU load low.

### 2.2 Constraints

- **Rate limits are hard boundaries.** The proxy server never exceeds the allowed request rate for any external API.
- **Greater London bounding box is fixed.** lat 51.2–51.7, lon -0.5–0.3. No dynamic region selection.
- **Desktop-first.** Mobile support is out of scope for V1.
- **No persistent storage.** All data is in-memory. No IndexedDB, no localStorage for transport data.
- **Single deployment target.** The app runs as a static frontend + a lightweight Express server. No container orchestration, no microservices.

### 2.3 Non-Goals (V1)

- Mobile support (touch controls, responsive layout)
- Performance settings menu (low/medium/high quality)
- Keyboard shortcuts
- Accessibility (screen reader, keyboard navigation)
- LOD (Level of Detail) for buildings
- Multi-city support
- Historical data playback
- Social sharing / export
- WebSocket or real-time push

---

## 3. System Architecture

### 3.1 Frontend Architecture

**Responsibilities:**
- Render the 3D scene (buildings, terrain, transport layers)
- Manage camera controls (orbit, zoom, pan, presets)
- Handle user interactions (hover, click, layer toggles)
- Display UI overlays (loading screen, tooltips, jamcam, layer toggles)
- Coordinate data fetching via backend API proxy
- Manage application state via Zustand

**Components:**

```
App (root)
├── LoadingScreen (full-screen, dismisses on ready)
├── SceneProvider (R3F Canvas + PostProcessing)
│   ├── OrbitControls (damping, auto-rotate)
│   ├── AmbientLight + DirectionalLight
│   ├── UnrealBloomPass (strength 1.5, radius 0.4, threshold 0.6)
│   ├── GroundPlane (Y=0, dark surface)
│   ├── ThamesWater (Y=-0.5, dark blue surface)
│   ├── Buildings (4 instanced meshes by height tier)
│   ├── Roads (grey line segments)
│   ├── TubeLines (13 colored neon lines + animated dots)
│   ├── BusMarkers (red capsules on road geometry)
│   ├── TrainMarkers (yellow dots on rail geometry)
│   ├── PlaneMarkers (white capsules at altitude)
│   ├── ShipMarkers (teal capsules on water)
│   ├── RiverBoats (green dots on waterway)
│   └── StreetLamps (yellow glow markers)
├── LayerToggles (top-right button group)
├── Tooltip (appears on vehicle hover)
└── JamcamOverlay (appears on signal hover)
```

**State Management (Zustand):**

```typescript
interface Vehicle {
  id: string;
  type: VehicleType;
  route: string;
  registration?: string;
  destination: string;
  nextStops?: string[];
  eta?: number;          // seconds to next stop
  lat: number;
  lng: number;
  altitude?: number;     // meters (planes only)
  speed?: number;        // knots (ships) or m/s (planes)
  heading?: number;      // degrees
  vertRate?: number;     // m/min (planes only)
  delay?: number;        // seconds (trains only)
  status?: 'on_time' | 'delayed' | 'cancelled';
  timestamp: number;     // Unix epoch of data fetch
}

interface TransportData {
  buses: Vehicle[];
  tubes: Vehicle[];
  trains: Vehicle[];
  planes: Vehicle[];
  ships: Vehicle[];
  riverBoats: Vehicle[];
  jamcams: Jamcam[];
  lastUpdated: Record<VehicleType, number>;
  loading: boolean;
  error: string | null;
  mapLoaded: boolean;
}

interface AppState extends TransportData {
  selectedVehicle: Vehicle | null;
  hoveredVehicle: Vehicle | null;
  hoveredSignal: Signal | null;
  visibleLayers: Set<VehicleType>;
  cameraPreset: 'default' | 'top-down' | 'close-up';
}
```

**Rendering Architecture:**
- `@react-three/fiber` Canvas is the root of all 3D rendering
- All scene objects are React components that declaratively describe the Three.js scene graph
- `requestAnimationFrame` drives the animation loop via R3F's `useFrame` hook
- Post-processing (bloom) is applied as a full-screen pass after the scene renders

### 3.2 Backend Architecture

**Responsibilities:**
- Proxy requests to external APIs (TfL, Darwin, ADSBx, AIS Hub)
- Serve static map data (GeoJSON files)
- Serve dummy data during development
- Enforce rate limits per data source
- Cache responses in memory with TTL
- Normalize external API responses into a consistent internal format

**API Proxy Design:**

The proxy server uses Express with route handlers per data type. Each handler follows this pattern:

```
Request → Check Cache → (if stale) → Fetch External API → Normalize → Cache → Response
```

**Caching:**
- In-memory `Map<string, CacheEntry>` where CacheEntry = `{ data, expiresAt, fetching }`
- TTL per data type (configured in constants)
- Stale-while-revalidate: if cache is expired but `fetching` is true, return stale data
- Cache is cold-started on server boot (first request triggers fetch)

**External API Integration:**
- Each data source has a dedicated fetcher module (`src/data/fetchers/tfl.ts`, `darwin.ts`, `adsbx.ts`, `ais.ts`)
- Fetchers return normalized data in the internal format
- Fetchers handle auth (API keys, usernames) from `process.env`
- Fetchers include error handling (network errors, HTTP errors, rate limits)

### 3.3 Data Layer

**Static Data:**
- Overture buildings: Pre-processed GeoJSON in `public/map-data/buildings.geojson`
- OSM geometry: Pre-processed GeoJSON in `public/map-data/geometry.geojson`
- Loaded once at app startup, cached in Zustand store
- Never re-fetched during runtime

**Real-Time Data:**
- Transport data: Fetched from backend proxy at configured intervals
- Jamcams: Fetched from TfL API at 60s interval
- All real-time data flows through the cache layer

**Transformation Pipeline:**

```
External API Response
  → Fetcher (auth, HTTP request)
  → Normalizer (field mapping, type conversion)
  → Validator (lat/lng bounds, required fields)
  → Cache (store with TTL)
  → Store (Zustand state update)
  → Renderer (Three.js mesh update)
```

---

## 4. Component Architecture

### 4.1 Rendering Engine

| Property | Value |
|----------|-------|
| **Responsibility** | Render the 3D scene using Three.js via React Three Fiber |
| **Inputs** | Zustand store (vehicle positions, visible layers), camera position |
| **Outputs** | Rendered WebGL canvas |
| **Dependencies** | Three.js, @react-three/fiber, @react-three/drei, @react-three/postprocessing |
| **Failure behavior** | If Three.js fails to initialize, show error overlay with message |

**Scene Graph:**
```
Canvas
├── PerspectiveCamera (position, fov, controls)
├── OrbitControls (damping: 0.05, autoRotate: false)
├── AmbientLight (color: 0x404060, intensity: 0.3)
├── DirectionalLight (color: 0xffeedd, intensity: 0.5)
├── EffectComposer
│   └── UnrealBloomPass (resolution: 256, strength: 1.5, radius: 0.4, threshold: 0.6)
├── GroundPlane (Y=0, size: 40000x40000)
├── ThamesWater (Y=-0.5, size: 30000x5000, animated)
├── Buildings (4 InstancedMesh groups by height)
├── Roads (LineSegments)
├── TubeLines (13 LineSegments + 13 InstancedMesh dots)
├── BusMarkers (InstancedMesh capsules)
├── TrainMarkers (InstancedMesh spheres)
├── PlaneMarkers (InstancedMesh capsules)
├── ShipMarkers (InstancedMesh capsules)
├── RiverBoats (InstancedMesh spheres)
└── StreetLamps (InstancedMesh small boxes)
```

### 4.2 Transport Layers

Each transport layer follows the same pattern:

| Property | Value |
|----------|-------|
| **Responsibility** | Render vehicles of one type, update positions each frame |
| **Inputs** | Vehicle array from Zustand store, route geometry |
| **Outputs** | Three.js mesh instances |
| **Dependencies** | Store, geo utilities, inference engine |
| **Failure behavior** | Render empty layer (no vehicles visible) |

**Per-layer specifics:**

| Layer | Mesh Type | Color | Orientation | Elevation |
|-------|-----------|-------|-------------|-----------|
| Tube | InstancedMesh spheres | Line color | N/A (dots) | Y=0.3 (above ground) |
| Bus | InstancedMesh capsules | Red | Forward along road | Y=0.1 (on road) |
| Train | InstancedMesh spheres | Yellow | N/A (dots) | Y=0.3 (above ground) |
| Plane | InstancedMesh capsules | White | By heading | Y=altitude (API) |
| Ship | InstancedMesh capsules | Teal | By heading | Y=-0.5 (in river) |
| River Boat | InstancedMesh spheres | Green | N/A (dots) | Y=-0.3 (on water) |

### 4.3 Inference Engine

| Property | Value |
|----------|-------|
| **Responsibility** | Calculate vehicle position along route geometry based on ETA and time elapsed |
| **Inputs** | Route geometry (array of lat/lng points), ETA (seconds), current time |
| **Outputs** | { lat, lng, heading } — position and orientation on the route |
| **Dependencies** | geo.ts (coordinate conversion), constants.ts |
| **Failure behavior** | Return last known position, log warning |

**Algorithm:**
```
1. Given route geometry [p0, p1, p2, ..., pn]
2. Calculate total route distance (sum of segment lengths)
3. Calculate progress: progress = elapsed_time / total_time
4. Find the segment where progress falls
5. Interpolate within that segment
6. Return position and heading (direction to next point)
```

### 4.4 Map Data Pipeline

| Property | Value |
|----------|-------|
| **Responsibility** | Load, transform, and serve map data (buildings + geometry) |
| **Inputs** | Overture Parquet tiles (downloaded), Overpass API response |
| **Outputs** | GeoJSON FeatureCollections served to frontend |
| **Dependencies** | parquetjs (Node.js), osmtogeojson (Node.js) |
| **Failure behavior** | If tiles are missing, render empty scene (no buildings) |

**Pipeline:**
```
Overture S3 → Download Parquet tiles → Convert to GeoJSON → Save to public/map-data/
Overpass API → Fetch geometry → Convert to GeoJSON → Save to public/map-data/
Frontend → Load GeoJSON files → Parse → Build Three.js geometries
```

### 4.5 API Clients

| Property | Value |
|----------|-------|
| **Responsibility** | Fetch data from external APIs, normalize responses |
| **Inputs** | None (fetchers are stateless, called with parameters) |
| **Outputs** | Normalized data in internal format |
| **Dependencies** | Node.js fetch, process.env for credentials |
| **Failure behavior** | Return empty array, log error, cache layer returns stale data |

**Per-client specifics:**

| Client | Endpoint | Auth | Normalization |
|--------|----------|------|---------------|
| TfL | Multiple endpoints | API key header | Map TfL fields → Vehicle interface |
| Darwin | /departure-board/{code} | API key header | Map Darwin fields → Vehicle interface |
| ADSBx | /area?lat=&lon=&rad= | x-api-key header | Map ADSBx fields → Vehicle interface |
| AIS Hub | /ws.php?username=... | Username query param | Map AIS fields → Vehicle interface |

### 4.6 Cache Layer

| Property | Value |
|----------|-------|
| **Responsibility** | Store fetched data with TTL, serve stale data while re-fetching |
| **Inputs** | Key (data type), TTL (seconds) |
| **Outputs** | Cached data or null |
| **Dependencies** | None (pure in-memory Map) |
| **Failure behavior** | Cache miss → fetch fresh data |

**Cache entry structure:**
```typescript
interface CacheEntry<T> {
  data: T;
  expiresAt: number;  // Unix timestamp
  fetching: boolean;  // Is a fetch currently in progress?
}
```

**TTL Configuration:**
```typescript
const CACHE_TTL: Record<VehicleType | 'jamcam', number> = {
  tube: 30,
  bus: 30,
  train: 30,
  plane: 10,
  ship: 60,
  riverBoat: 30,
  jamcam: 60,
};
```

### 4.7 UI Layer

| Property | Value |
|----------|-------|
| **Responsibility** | Render HTML overlays (loading screen, tooltips, jamcam, layer toggles) |
| **Inputs** | Zustand store (hovered vehicle, hovered signal, loading state) |
| **Outputs** | HTML elements overlaid on the 3D canvas |
| **Dependencies** | Zustand store, Tooltip component, JamcamOverlay component |
| **Failure behavior** | UI degrades gracefully (no tooltips, no overlays) |

---

## 5. Data Architecture

### 5.1 Domain Entities

**Vehicle** (shared by all transport types):
```
Vehicle
├── identity
│   ├── id: string          // Unique identifier (route + index)
│   ├── type: VehicleType   // 'tube' | 'bus' | 'train' | 'plane' | 'ship' | 'riverBoat'
│   ├── route: string       // Route number/name (e.g., "N343", "Central", "BAW123")
│   └── registration?: string  // Vehicle registration (buses, ships)
├── position
│   ├── lat: number         // Latitude (WGS84)
│   ├── lng: number         // Longitude (WGS84)
│   └── altitude?: number   // Altitude in meters (planes only)
├── movement
│   ├── heading?: number    // Direction in degrees (0=N, 90=E)
│   ├── speed?: number      // Speed (m/s for planes, knots for ships)
│   └── vertRate?: number   // Vertical rate m/min (planes only)
├── metadata
│   ├── destination: string // Final destination
│   ├── nextStops?: string[] // Upcoming stops/stations
│   ├── eta?: number        // Seconds to next stop
│   ├── delay?: number      // Seconds of delay (trains only)
│   └── status?: string     // 'on_time' | 'delayed' | 'cancelled'
└── timestamp
    └── timestamp: number   // Unix epoch of data fetch
```

**Jamcam:**
```
Jamcam
├── id: string              // TFL jamcam ID
├── name: string            // Camera name (e.g., "Piccadilly Circus")
├── lat: number             // Camera position
├── lng: number             // Camera position
├── available: boolean      // Is the camera working?
├── imageUrl: string        // Static snapshot URL
├── videoUrl: string        // Live video URL
└── view: string            // Camera view description
```

**Signal (Street Lamp):**
```
Signal
├── id: string              // Unique identifier (index-based)
├── lat: number             // Position on road
├── lng: number             // Position on road
├── jamcamId?: string       // Associated jamcam ID (if any)
└── roadSegment: string     // Road identifier
```

**RouteGeometry:**
```
RouteGeometry
├── id: string              // Route identifier
├── type: GeometryType      // 'road' | 'rail' | 'waterway' | 'busRoute'
├── name: string            // Route name
└── points: GeoPoint[]      // Ordered array of {lat, lng}
```

### 5.2 TypeScript Interfaces

Full interfaces are defined in `src/store/types.ts`. Key interfaces:

```typescript
type VehicleType = 'tube' | 'bus' | 'train' | 'plane' | 'ship' | 'riverBoat';
type GeometryType = 'road' | 'rail' | 'waterway' | 'busRoute';

interface Vehicle { ... }  // See section 5.1
interface Jamcam { ... }   // See section 5.1
interface Signal { ... }   // See section 5.1
interface RouteGeometry { ... }  // See section 5.1

interface TransportData {
  buses: Vehicle[];
  tubes: Vehicle[];
  trains: Vehicle[];
  planes: Vehicle[];
  ships: Vehicle[];
  riverBoats: Vehicle[];
  jamcams: Jamcam[];
  lastUpdated: Record<VehicleType | 'jamcam', number>;
  loading: boolean;
  error: string | null;
  mapLoaded: boolean;
}

interface AppState extends TransportData {
  selectedVehicle: Vehicle | null;
  hoveredVehicle: Vehicle | null;
  hoveredSignal: Signal | null;
  visibleLayers: Set<VehicleType>;
  cameraPreset: 'default' | 'top-down' | 'close-up';
}
```

### 5.3 Data Ownership

| Data | Owned By | Updated By |
|------|----------|------------|
| Vehicles | Zustand store (frontend) | Backend proxy (via API) |
| Map data | Zustand store (frontend) | Static files (pre-processed) |
| Cache | Backend proxy (Express) | Proxy fetchers |
| UI state | Zustand store (frontend) | User interaction |

### 5.4 Data Lifecycle

```
1. Server boot → Load static map data from files
2. Server boot → Start polling timers per data type
3. Poll timer fires → Check cache → Fetch if stale → Normalize → Cache → Proxy → Frontend
4. Frontend receives data → Update Zustand store → R3F re-renders
5. R3F useFrame → Update vehicle positions → GPU renders
```

### 5.5 Update Frequency

| Data Type | Poll Interval | TTL | Notes |
|-----------|--------------|-----|-------|
| Tube | 30s | 30s | Burst arrivals at stations |
| Bus | 30s | 30s | Mix of GPS + inferred |
| Train | 30s | 30s | Inferred from departures |
| Plane | 10s | 10s | Max allowed by ADSBx |
| Ship | 60s | 60s | Max allowed by AIS Hub |
| River Boat | 30s | 30s | Inferred from arrivals |
| Jamcam | 60s | 60s | Static data |

---

## 6. Runtime Data Flow

### 6.1 Tube Data Flow

```
TfL API /Metro/Live/Arrivals/{stationId}
  → Backend TfL Fetcher (auth: API key)
  → Normalizer (map lineName, destination, minutesToStation → Vehicle[])
  → Validator (check lat/lng bounds)
  → Cache (TTL: 30s)
  → Proxy GET /api/tube/arrivals
  → Frontend fetch()
  → Zustand store.tubes update
  → TubeLines component re-renders
  → useFrame updates dot positions along line geometry
  → GPU renders neon line + moving dots
```

### 6.2 Bus Data Flow

```
TfL API /Bus/RealTime + /Metro/Live/Arrivals/{stationId}
  → Backend TfL Fetcher (auth: API key)
  → Normalizer (GPS buses: direct position; inferred: calculate from ETA + route geometry)
  → Validator (check lat/lng, snap to road geometry)
  → Cache (TTL: 30s)
  → Proxy GET /api/buses
  → Frontend fetch()
  → Zustand store.buses update
  → BusMarkers component re-renders
  → useFrame updates capsule positions along road geometry
  → GPU renders red capsules on roads
```

### 6.3 Train Data Flow

```
Darwin DBRS /departure-board/{NaptocCode}
  → Backend Darwin Fetcher (auth: API key)
  → Normalizer (map TargetArrival, ExpectedArrival, DestinationName → Vehicle[])
  → Validator (check lat/lng, snap to rail geometry)
  → Cache (TTL: 30s)
  → Proxy GET /api/trains
  → Frontend fetch()
  → Zustand store.trains update
  → TrainMarkers component re-renders
  → useFrame updates dot positions along rail geometry
  → GPU renders yellow dots on rails
```

### 6.4 Plane Data Flow

```
ADSBx v2 /area?lat=51.5&lon=-0.1&rad=25
  → Backend ADSBx Fetcher (auth: x-api-key header, Accept-Encoding: gzip)
  → Normalizer (map callsign, lat, lng, baro_altitude, true_track, speed_ground → Vehicle[])
  → Validator (check lat/lng, altitude > 0, is_on_ground === false)
  → Cache (TTL: 10s)
  → Proxy GET /api/planes
  → Frontend fetch()
  → Zustand store.planes update
  → PlaneMarkers component re-renders
  → useFrame updates capsule positions (velocity * dt)
  → GPU renders white capsules at altitude
```

### 6.5 Ship Data Flow

```
AIS Hub /ws.php?username=...&latmin=...&latmax=...&lonmin=...&lonmax=...
  → Backend AIS Fetcher (auth: username query param)
  → Normalizer (map NAME, MMSI, LATITUDE, LONGITUDE, SOG, HEADING → Vehicle[])
  → Validator (check lat/lng, SOG >= 0, HEADING 0-360)
  → Cache (TTL: 60s)
  → Proxy GET /api/ships
  → Frontend fetch()
  → Zustand store.ships update
  → ShipMarkers component re-renders
  → useFrame updates capsule positions (SOG * dt)
  → GPU renders teal capsules on water
```

### 6.6 Jamcam Data Flow

```
TfL API /Place/Type/JamCam
  → Backend TfL Fetcher (auth: API key)
  → Normalizer (map commonName, lat, lon, imageUrl, videoUrl → Jamcam[])
  → Validator (check lat/lng, available === true)
  → Cache (TTL: 60s)
  → Proxy GET /api/jamcams
  → Frontend fetch()
  → Zustand store.jamcams update
  → StreetLamps component assigns nearest jamcam to each lamp
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
├── Geometry (static):
│   ├── GroundPlane (PlaneGeometry, Y=0)
│   ├── ThamesWater (PlaneGeometry, Y=-0.5, animated)
│   ├── Roads (LineSegments from OSM geometry)
│   ├── TubeLines (13 LineSegments, one per line)
│   ├── RailLines (LineSegments from OSM geometry)
│   ├── Waterway (LineSegments for Thames)
│   └── StreetLamps (InstancedMesh, ~500 instances)
├── Geometry (dynamic, updated each frame):
│   ├── Buildings (4 InstancedMesh, grouped by height tier)
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
        dpr={[1, 2]}  // Cap at 2x
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
- Group by height tier: [0-10m], [10-30m], [30-60m], [60m+]
- Each group becomes one `InstancedMesh`
- ~4 instanced meshes total for all 50k-100k buildings

**Vehicles:**
- One `InstancedMesh` per transport type
- ~6 instanced meshes total for all vehicles
- Each instance has its own transform matrix (position + rotation) and color

**Total draw calls:** ~212 (4 buildings + 6 vehicles + ~200 lines/roads/lamps)

### 7.4 LOD Strategy

**V1 (current):** No LOD. All geometry is rendered at full detail. Frustum culling ensures only visible elements are drawn.

**Future enhancement:** Implement LOD for buildings:
- LOD 0 (close): Full building geometry with edges
- LOD 1 (medium): Simplified box
- LOD 2 (far): Silhouette only
- LOD 3 (very far): Not rendered

### 7.5 Coordinate System

| Axis | Meaning | Range |
|------|---------|-------|
| X | Easting (Web Mercator) | -5000 to +5000 meters |
| Y | Altitude/Elevation | -0.5 (river) to +5000 (air) |
| Z | Northing (Web Mercator) | -5000 to +5000 meters |

**Origin:** London center (51.5074°N, -0.1278°W) → (0, 0, 0)

**Conversion:** Web Mercator projection (EPSG:3857) converts lat/lng to X/Z in meters.

### 7.6 Animation Loop

All animation happens in R3F's `useFrame` hook:

```typescript
useFrame(({ clock }) => {
  const delta = clock.getDelta();
  const elapsed = clock.getElapsedTime();

  // Update bus positions along road geometry
  buses.forEach(bus => {
    bus.position = interpolateAlongGeometry(bus.routeGeometry, bus.eta, elapsed);
    bus.rotation.y = calculateHeading(bus.routeGeometry, bus.position);
  });

  // Update train positions along rail geometry
  trains.forEach(train => { ... });

  // Update plane positions (velocity-based)
  planes.forEach(plane => {
    plane.position.x += plane.speed * Math.cos(plane.heading) * delta;
    plane.position.z += plane.speed * Math.sin(plane.heading) * delta;
    plane.position.y = plane.altitude;
  });

  // Update ship positions (speed-based)
  ships.forEach(ship => { ... });

  // Update instance matrices
  updateInstancedMeshes();
});
```

---

## 8. Backend Architecture

### 8.1 Routes

```typescript
// server/proxy.ts
const router = express.Router();

// Transport data
router.get('/api/tube/arrivals', tflController.getTubeArrivals);
router.get('/api/buses', tflController.getBuses);
router.get('/api/trains', darwinController.getTrains);
router.get('/api/planes', adsbxController.getPlanes);
router.get('/api/ships', aisController.getShips);
router.get('/api/river-boats', tflController.getRiverBoats);
router.get('/api/jamcams', tflController.getJamcams);

// Map data (served from static files)
router.get('/api/map/buildings', mapController.getBuildings);
router.get('/api/map/geometry', mapController.getGeometry);

// Health check
router.get('/api/health', (req, res) => res.json({ status: 'ok' }));
```

### 8.2 Services

| Service | File | Responsibility |
|---------|------|----------------|
| TfLController | `server/proxy/tfl.ts` | Tube, bus, river, jamcam data |
| DarwinController | `server/proxy/darwin.ts` | National rail data |
| ADSBxController | `server/proxy/adsbx.ts` | Aircraft data |
| AISController | `server/proxy/ais.ts` | Vessel data |
| MapController | `server/proxy/map.ts` | Static map data |
| Cache | `server/cache.ts` | In-memory TTL cache |
| RateLimiter | `server/rateLimiter.ts` | Token bucket rate limiter |

### 8.3 Cache Strategy

```typescript
// server/cache.ts
class Cache {
  private store = new Map<string, CacheEntry>();

  get<T>(key: string): T | null {
    const entry = this.store.get(key);
    if (!entry) return null;
    if (entry.expiresAt < Date.now()) {
      // Stale but fetching in progress
      if (entry.fetching) return entry.data;
      // Stale and not fetching, remove
      this.store.delete(key);
      return null;
    }
    return entry.data as T;
  }

  set<T>(key: string, data: T, ttl: number): void {
    this.store.set(key, {
      data,
      expiresAt: Date.now() + ttl * 1000,
      fetching: false,
    });
  }

  async getOrFetch<T>(key: string, fetchFn: () => Promise<T>, ttl: number): Promise<T> {
    // Check cache first
    const cached = this.get<T>(key);
    if (cached) return cached;

    // Check if fetch is in progress
    const entry = this.store.get(key);
    if (entry?.fetching) {
      // Wait for existing fetch to complete
      return new Promise<T>((resolve) => {
        const check = setInterval(() => {
          const e = this.store.get(key);
          if (e && !e.fetching) {
            clearInterval(check);
            resolve(e.data as T);
          }
        }, 100);
      });
    }

    // Start new fetch
    this.store.set(key, { data: null as any, expiresAt: 0, fetching: true });
    try {
      const data = await fetchFn();
      this.set(key, data, ttl);
      return data;
    } catch (err) {
      this.store.delete(key);
      throw err;
    }
  }
}
```

### 8.4 Error Handling

| Error Type | Handling |
|------------|----------|
| Network timeout | Retry once after 2s, then return empty array |
| HTTP 429 (rate limit) | Wait for rate limit window, retry once |
| HTTP 5xx | Retry once after 1s, then return empty array |
| Invalid response | Log error, return empty array |
| Missing required fields | Log warning, skip that vehicle |
| Coordinates out of bounds | Log warning, skip that vehicle |

### 8.5 Retries

- **Max retries:** 1 per request
- **Retry delay:** 1s for 5xx, 2s for network timeout
- **No exponential backoff** (simple linear retry is sufficient for this scale)

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
    └── node server/index.ts (port 3001) — Backend proxy server

Vite config proxy:
  server.proxy['/api'] = 'http://localhost:3001'

Environment:
  .env.local — API keys (gitignored)
  .env.example — Template (committed)
```

**Dependencies:**
```json
{
  "devDependencies": {
    "concurrently": "^9.0.0"
  },
  "scripts": {
    "dev": "concurrently \"npm run dev:frontend\" \"npm run dev:backend\"",
    "dev:frontend": "vite",
    "dev:backend": "node server/index.ts"
  }
}
```

### 9.2 Production Deployment

**Option A: Static hosting + separate backend**
- Frontend: Vercel, Netlify, or Cloudflare Pages (static files)
- Backend: Railway, Render, or Fly.io (Express server)
- Map data: Embedded in frontend build (static files)

**Option B: Single server**
- One server serves both frontend and backend
- Express serves static files from `dist/` + API routes
- Deployed via Docker or direct to VPS

**Recommended for V1:** Option B (simpler, fewer moving parts)

### 9.3 Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `TFL_API_KEY` | Yes | TfL API key |
| `DARWIN_API_KEY` | Yes | Darwin DBRS API key |
| `ADSBX_API_KEY` | Yes | ADSBx API key |
| `AIS_HUB_USERNAME` | Yes | AIS Hub username |
| `PORT` | No (default: 3001) | Backend server port |

### 9.4 Static Assets

| Asset | Location | Generated By |
|-------|----------|--------------|
| Buildings GeoJSON | `public/map-data/buildings.geojson` | `scripts/download-map-data.ts` |
| Geometry GeoJSON | `public/map-data/geometry.geojson` | `scripts/fetch-osm-geometry.ts` |
| Built frontend | `dist/` | `vite build` |

---

## 10. Security Architecture

### 10.1 API Key Handling

| Layer | Handling |
|-------|----------|
| Storage | `.env.local` (gitignored), read via `process.env` |
| Transmission | HTTPS only for all external API calls |
| Frontend | Never exposed to browser — all API calls go through backend proxy |
| Logging | API keys are never logged or printed |

### 10.2 Secrets Management

- **Development:** `.env.local` file (never committed)
- **Production:** Environment variables set by hosting platform
- **No secret managers** (overkill for this project)

### 10.3 Input Validation

All external API responses are validated before being stored:

```typescript
function validateVehicle(data: unknown): Vehicle | null {
  if (!data || typeof data !== 'object') return null;
  const obj = data as Record<string, unknown>;

  // Required fields
  if (typeof obj.lat !== 'number' || typeof obj.lng !== 'number') return null;

  // Bounds check (Greater London)
  if (obj.lat < 51.0 || obj.lat > 52.0 || obj.lng < -1.0 || obj.lng > 1.0) return null;

  // Type-specific validation
  if (type === 'plane' && (typeof obj.altitude !== 'number' || obj.altitude < 0)) return null;
  if (type === 'ship' && (typeof obj.speed !== 'number' || obj.speed < 0)) return null;

  return mapToVehicle(obj, type);
}
```

### 10.4 External API Failures

| Scenario | Behavior |
|----------|----------|
| API returns 5xx | Retry once, then return empty array (graceful degradation) |
| API times out | Retry once, then return empty array |
| API returns malformed JSON | Log error, return empty array |
| API key invalid | Log error, show warning in console, return empty array |
| Rate limit exceeded | Wait and retry, log warning |

---

## 11. Observability

### 11.1 Logging

**Backend:**
- `console.log` for normal operations (data fetches, cache hits/misses)
- `console.warn` for recoverable issues (stale data, missing fields)
- `console.error` for failures (network errors, parse errors)

**Frontend:**
- `console.log` for data updates (store changes)
- `console.warn` for rendering issues (missing geometry, invalid positions)
- `console.error` for crashes (Three.js errors, store corruption)

**Log format:** Simple string messages with context prefix.
```
[TFL] Fetched 247 stations in 340ms
[ADSBX] Fetched 42 aircraft in 1200ms
[Cache] HIT tube: 30s remaining
[Cache] MISS plane: fetching...
[Inference] Bus N343: position 0.73 along route (ETA 95s)
```

### 11.2 Metrics

**V1 (no metrics collection):** No external monitoring. Performance is verified visually.

**Future enhancement:** Add simple metrics:
- API response times (p50, p95, p99)
- Cache hit rate
- Vehicle count per layer
- Frame rate (FPS)

### 11.3 Debugging Approach

1. **Visual debugging:** Toggle layer visibility, zoom to specific areas, check vehicle positions
2. **Console logging:** Backend logs show fetch times, cache status, errors
3. **Store inspection:** `useStore.getState()` in browser console to inspect current state
4. **Network tab:** Frontend Network tab shows API responses
5. **DevTools:** React DevTools for component tree, Zustand middleware for store debugging

---

## 12. Architecture Decision Records (ADRs)

The following decisions deserve formal ADRs:

### ADR-001: Dummy-First Development
- **Status:** Accepted
- **Context:** API keys are required for all external data sources. Registration takes time. Rate limits constrain testing.
- **Decision:** Build and test the entire frontend with dummy data. Real API integration is a drop-in replacement.
- **Consequences:** Faster development iteration. Visual feedback immediately. API integration is a focused final phase.

### ADR-002: Instanced Meshes for All Vehicles
- **Status:** Accepted
- **Context:** 6 transport types with hundreds of vehicles. Individual meshes would cause thousands of draw calls.
- **Decision:** One `InstancedMesh` per transport type. Per-instance colors via `setColorAt()`.
- **Consequences:** ~6 draw calls for all vehicles. Minimal GPU load. Slightly less flexibility per vehicle (can't have unique materials).

### ADR-003: Geometry-Constrained Animation
- **Status:** Accepted
- **Context:** Buses and trains must appear on roads/tracks, not fly over buildings.
- **Decision:** All vehicle positions are projected onto the nearest point of the road/rail geometry array.
- **Consequences:** Vehicles always appear on correct paths. Requires OSM geometry data. More complex than free-space interpolation.

### ADR-004: Pre-processed Map Data
- **Status:** Accepted
- **Context:** Overture provides data as Parquet tiles. Parsing in the browser requires WASM.
- **Decision:** Download and convert Parquet → GeoJSON at build time. Serve static GeoJSON files.
- **Consequences:** Zero runtime parsing cost. Smaller browser bundle. Requires build script. Data can drift (Overture updates periodically).

### ADR-005: Polling Over WebSockets
- **Status:** Accepted
- **Context:** External APIs are HTTP-based. No WebSocket endpoints available.
- **Decision:** HTTP polling with configurable intervals. No WebSocket connections.
- **Consequences:** Simpler architecture. Higher latency than push (but acceptable for this use case). More HTTP requests than necessary.

### ADR-006: Express Proxy Over Pure Frontend
- **Status:** Accepted
- **Context:** External APIs don't support CORS. API keys must not be exposed in browser.
- **Decision:** Express proxy server handles all external API calls. Frontend calls proxy endpoints.
- **Consequences:** Solves CORS and key exposure. Adds a server process. Slightly more complex dev setup.

### ADR-007: Bloom Post-Processing for Neon Glow
- **Status:** Accepted
- **Context:** Reference screenshots show glowing neon transport lines.
- **Decision:** `UnrealBloomPass` from @react-three/postprocessing.
- **Consequences:** Simple implementation. Looks great. One extra full-screen pass (~1-2ms on modern GPU).

### ADR-008: TDD for Logic Layer Only
- **Status:** Accepted
- **Context:** Project is mostly integration-heavy (3D rendering, API calls, UI). TDD on components is brittle.
- **Decision:** TDD for pure logic modules (geo.ts, routeInterpolation.ts, cache.ts, inference.ts). Implement-and-test for components and API fetchers.
- **Consequences:** Tests focus on what's testable and valuable. No brittle DOM/Three.js assertions. Faster test development.

---

## 13. Assumptions

1. **Overture tiles are available** for the Greater London bounding box. If not, we fall back to a smaller area.
2. **Overpass API returns complete geometry** for all major roads, rail lines, and waterways in Greater London. Some minor routes may be missing.
3. **TfL API keys are obtained** before API integration phase. Registration is free and instant.
4. **ADSBx free tier provides sufficient coverage** for London airspace. If not, we may need to expand the bounding box radius.
5. **AIS Hub provides Thames coverage.** If vessel density is low, we may need to expand the bounding box.
6. **Three.js instanced meshes support per-instance colors** via `setColorAt()`. This is documented behavior but should be verified.
7. **Web Mercator projection is accurate enough** for London-scale visualization. For km-scale distances, distortion is minimal.

## 14. Open Questions

1. **What if Overture tiles are too large?** Estimated 400-800MB. If this is unacceptable, we may need to tile and stream buildings.
2. **Should we cache GeoJSON on the backend or serve directly from static files?** Currently planned as static files. Backend caching adds complexity.
3. **What happens if a user has WebGL disabled?** We should detect this and show a fallback message.
4. **Should street lamps be placed on all roads or just major roads?** Currently planned for major roads only (motorway, trunk, primary).
5. **What is the minimum supported browser?** Three.js requires WebGL 1.0+. Chrome, Firefox, Safari, Edge should all work.

---

## 15. Glossary

| Term | Definition |
|------|------------|
| **ADSBx** | ADS-B Exchange, a global aircraft tracking network |
| **AIS** | Automatic Identification System, for vessel tracking |
| **Darwin DBRS** | National Rail's Departure Board Reporting System |
| **EPSG:3857** | Web Mercator projection, used for lat/lng → X/Z conversion |
| **GeoJSON** | JSON-based format for geographic data |
| **InstancedMesh** | Three.js class for rendering many objects with one draw call |
| **JAMCAM** | Transport for London's traffic camera system |
| **LOD** | Level of Detail, rendering simplified geometry at distance |
| **NaptocCode** | National Public Transport Access Code, used by Darwin |
| **Overture Maps** | Open-source map data project by Meta, Apple, Mapbox, OpenStreetMap |
| **Overpass API** | Read-only API for querying OpenStreetMap data |
| **Parquet** | Columnar storage format, used by Overture for tile data |
| **R3F** | React Three Fiber, declarative Three.js for React |
| **TfL** | Transport for London |
| **UnrealBloomPass** | Three.js post-processing effect for glow |
| **Web Mercator** | Map projection used by most web maps |
| **Zustand** | Lightweight React state management library |
