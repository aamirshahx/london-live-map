# London Live 3D Map — Product Requirements Document

## 1. Product Overview

**Product Name:** Zone One — A Live 3D Map of Central London

**Description:** A real-time, interactive 3D visualization of London showing every Tube train, bus, National Rail train, river boat, aircraft, and ship moving in real time. Transport without GPS (buses, trains) is inferred from arrival countdowns and animated along actual road/rail geometry.

**Tagline:** A live map of central London · every tube, bus, train, riverboat and aircraft moving in real time

**Target Audience:** Transit enthusiasts, urban planners, London residents, data visualization fans

**Platform:** Web browser (desktop-first, responsive)

---

## 2. Data Sources & API Specifications

### 2.1 TfL API (Tube, Bus, River)

| Field | Value |
|-------|-------|
| **Base URL** | `https://api.tfl.gov.uk` |
| **Auth** | Free API key (register at https://api.tfl.gov.uk/) |
| **Rate Limit** | 10 requests/second, 100,000 calls/day |
| **Key Endpoints** | See subsections below |

#### Tube (Metro Live Arrivals)
```
GET /Metro/Live/Arrivals/{stationId}
Response: [{
  "lineName": "Central",
  "destination": "Hainault",
  "minutesToStation": 3,
  "lineId": 7,
  "bearing": "270",
  "lat": 51.5224,
  "lon": -0.0534
}]
```
- Polling interval: **30s**
- Returns: arriving trains per station
- Inference: place each train on line geometry proportional to ETA

#### Bus (RealTime / Live Arrivals)
```
GET /Bus/RealTime
GET /Metro/Live/Arrivals/{stationId}
```
- Polling interval: **30s**
- Some buses have live GPS (RealTime API), others inferred from arrivals
- Inference: snap to road geometry, animate along route

#### River Bus
```
GET /River/StopPoints
GET /Metro/Live/Arrivals/{stationId}
```
- Polling interval: **30s**
- River boat positions inferred from arrival data
- Animate along Thames waterway geometry

#### Jamcams
```
GET /Place/Type/JamCam
Response: [{
  "id": "JamCams_00001.07450",
  "commonName": "Piccadilly Circus",
  "lat": 51.5096,
  "lon": -0.13484,
  "additionalProperties": [{
    "key": "available", "value": "true"
  }, {
    "key": "imageUrl", "value": "https://s3-eu-west-1.amazonaws.com/jamcams.tfl.gov.uk/00001.07450.jpg"
  }, {
    "key": "videoUrl", "value": "https://s3-eu-west-1.amazonaws.com/jamcams.tfl.gov.uk/00001.07450.mp4"
  }, {
    "key": "view", "value": "Piccadilly Circus"
  }]
}]
```
- Polling interval: **60s** (static data, rarely changes)
- Returns: all jamcams in London with positions, images, video URLs
- Used to match street lamps to camera feeds

### 2.2 National Rail Darwin DBRS

| Field | Value |
|-------|-------|
| **Base URL** | `http://webapi.datatools.stack.com/darwin/v1` |
| **Auth** | Free account required (register at https://realtime.nationalrail.co.uk/DBRSLogin/) |
| **Rate Limit** | 100,000 credits/day |
| **Key Endpoint** | See below |

#### Departure Boards
```
GET /departure-board/{NaptocCode}?includeArrivals=true&limit=10
Response: [{
  "NaptocCode": "9100GLVC",
  "CommonName": "Victoria Station",
  "Arrivals": [{
    "LineName": "Southern",
    "DestinationName": "Brighton",
    "TargetArrival": "2026-06-18T14:30:00Z",
    "ExpectedArrival": "2026-06-18T14:32:00Z",
    "Bearing": "180"
  }]
}]
```
- Polling interval: **30s**
- Returns: scheduled/expected arrivals per station
- Inference: place each train on rail geometry proportional to ETA

### 2.3 ADSBx v2 API (Planes)

| Field | Value |
|-------|-------|
| **Base URL** | `https://gateway.adsbexchange.com/api/aircraft/v2` |
| **Auth** | Free API key via `x-api-key` header (register at https://www.adsbexchange.com/products/enterprise-api/) |
| **Rate Limit** | 1 request per 10 seconds |
| **Required Header** | `Accept-Encoding: gzip` |

#### Area Query
```
GET /area?lat=51.5&lon=-0.1&rad=25
Response: [{
  "icao24": "400b21",
  "callsign": "BAW123",
  "origin_country": "United Kingdom",
  "time_position": 1718712000,
  "last_contact": 1718712005,
  "longitude": -0.1234,
  "latitude": 51.5098,
  "baro_altitude": 3500,
  "geometry_altitude": 3480,
  "velocity": 180,
  "speed_ground": 175,
  "true_track": 270,
  "heading": 268,
  "vert_rate": -200,
  "squawk": "1234",
  "category": "A3",
  "is_on_ground": false,
  "mlat": [],
  "is_icao24_valid": true,
  "messages": 1234,
  "seen": 2,
  "rssi": -25
}]
```
- Polling interval: **10s** (max allowed by rate limit)
- Bounding box: lat 51.2–51.7, lon -0.5–0.3 (Greater London)
- Radius: 25km from London center (51.5074, -0.1278)
- Returns: all aircraft in area with position, altitude, speed, heading, callsign

### 2.4 AIS Hub (Ships)

| Field | Value |
|-------|-------|
| **Base URL** | `https://data.aishub.net/ws.php` |
| **Auth** | Registered username as `username` parameter (register at https://www.aishub.net/join-us) |
| **Rate Limit** | 1 request per minute (returns nothing if called more frequently) |
| **Formats** | JSON, XML, CSV |

#### Vessel Query
```
GET /ws.php?username=USERNAME&format=1&output=json&compress=3&latmin=51.4&latmax=51.55&lonmin=-0.2&lonmax=0.1
Response: [{
  "MMSI": 235006759,
  "TSTAMP": 1718712000,
  "LATITUDE": 51.4900,
  "LONGITUDE": -0.1200,
  "COG": 45,
  "SOG": 5,
  "HEADING": 42,
  "NAVSTAT": 0,
  "IMO": 9123456,
  "NAME": "ALMERE 4",
  "CALLSIGN": "PDMD",
  "TYPE": 70,
  "A": 28,
  "B": 4,
  "C": 0,
  "DRAUGHT": 1.0,
  "DEST": "TOWER BRIDGE",
  "ETA": 1718715600
}]
```
- Polling interval: **60s** (max allowed by rate limit)
- Bounding box: tidal Thames area (Greenwich to Tower Bridge + estuary)
- Returns: vessel position, speed, heading, name, dimensions, flag

### 2.5 Map Data

#### Overture Maps (3D Buildings)

| Field | Value |
|-------|-------|
| **Source** | https://docs.overturemaps.org |
| **Format** | Parquet tiles (downloaded and converted to GeoJSON) |
| **Coverage** | Greater London (lat 51.2–51.7, lon -0.5–0.3) |
| **Download URL** | `https://overturemaps-tiles-us-east-1.s3.amazonaws.com/2024-04-24.0/release/` |
| **Processing** | One-time: Parquet → GeoJSON, saved to `public/map-data/` |
| **Serving** | Static files loaded by frontend on startup |
| **Estimated Size** | 400–800MB total (downloaded once) |
| **Building Count** | ~50,000–100,000 buildings |

#### Overpass API (Route Geometry)

| Field | Value |
|-------|-------|
| **Base URL** | `https://overpass-api.de/api/interpreter` |
| **Auth** | None required |
| **Rate Limit** | ~1 request/second (be respectful) |
| **Query Type** | One-time fetch on startup, cached in app |
| **Estimated Size** | 50–100MB of geometry data |

**Overpass Query (executed once at build time):**
```
[out:json][timeout:120];
(
  way["railway"~"rail|light_rail|subway"](51.2,-0.5,51.7,0.3);
  way["route"="bus"](51.2,-0.5,51.7,0.3);
  way["waterway"="river"](51.2,-0.5,51.7,0.3);
  way["highway"~"motorway|trunk|primary"](51.2,-0.5,51.7,0.3);
  node["highway"="street_lamp"](51.2,-0.5,51.7,0.3);
);
out body;
> out qt;
```

Returns: ordered node arrays for each road/rail/waterway segment, with lat/lng coordinates.

---

## 3. Architecture

### 3.1 System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Frontend (Vite + React)                   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  Zustand Store                           │   │
│  │  - vehicles (all types: bus, tube, train, plane, ship)   │   │
│  │  - selectedVehicle / hoveredVehicle                      │   │
│  │  - mapData (buildings, geometry)                         │   │
│  │  - loading / error state                                 │   │
│  └──────────────────────────────────────────────────────────┘   │
│           │                              │                       │
│  ┌────────▼──────────┐      ┌───────────▼──────────┐           │
│  │  R3F 3D Scene     │      │  Tooltip UI (HTML)   │           │
│  │  - Buildings      │      │  - Transport info    │           │
│  │  - Thames water   │      │  - Jamcam overlay    │           │
│  │  - Neon lines     │      │  - Layer toggles     │           │
│  │  - Vehicle markers│      │  - Loading screen    │           │
│  └───────────────────┘      └──────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
           │                              │
           │  fetch() /api/*              │  fetch() /api/*
┌──────────▼──────────────────────────────▼───────────────────────┐
│                    Backend (Express + Node.js)                   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────┐       │
│  │          In-Memory Cache (TTL per data type)          │       │
│  │  - Tube: 30s  - Bus: 30s  - Rail: 30s               │       │
│  │  - Planes: 10s  - Ships: 60s  - Jamcams: 60s         │       │
│  └──────────────────────────────────────────────────────┘       │
│           │                              │                       │
│  ┌────────▼──────────┐      ┌───────────▼──────────┐           │
│  │  Data Fetchers    │      │  Dummy Data (dev)    │           │
│  │  (staged for API  │      │  (active during dev) │           │
│  │   integration)    │      │                     │           │
│  │  - tfl.ts         │      │  - buses.json        │           │
│  │  - darwin.ts      │      │  - planes.json       │           │
│  │  - opensky.ts     │      │  - ships.json        │           │
│  │  - ais.ts         │      │  - tube.json         │           │
│  └───────────────────┘      └──────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Technology Stack

| Layer | Technology | Reason |
|-------|-----------|--------|
| **Frontend Framework** | React 19 + TypeScript | Mature ecosystem, R3F integration |
| **3D Rendering** | Three.js + @react-three/fiber + @react-three/drei | Declarative 3D in React |
| **Post-Processing** | @react-three/postprocessing (UnrealBloomPass) | Neon glow effect |
| **State Management** | Zustand | Lightweight, works well with R3F |
| **Build Tool** | Vite | Fast HMR, good R3F support |
| **Backend** | Express + Node.js | Simple proxy server, solves CORS |
| **Concurrency** | concurrently | Run Vite + Express in dev |
| **Map Data Format** | GeoJSON (pre-processed from Parquet) | Easy to consume in browser |
| **Data Format** | JSON | Standard, easy to parse |

### 3.3 Directory Structure

```
london-live-map/
├── src/
│   ├── main.tsx                    # App entry point
│   ├── App.tsx                     # Root component (loading screen + scene)
│   ├── index.css                   # Global styles
│   ├── App.css                     # App-specific styles
│   │
│   ├── store/
│   │   ├── index.ts                # Zustand store (all state)
│   │   ├── types.ts                # Shared TypeScript types
│   │   └── selectors.ts            # Derived state helpers
│   │
│   ├── data/
│   │   ├── fetchers/
│   │   │   ├── tfl.ts              # TfL API client (tube, bus, river, jamcams)
│   │   │   ├── darwin.ts           # Darwin DBRS client (national rail)
│   │   │   ├── adsbx.ts            # ADSBx v2 client (planes)
│   │   │   └── ais.ts              # AIS Hub client (ships)
│   │   ├── inference.ts            # Position inference engine
│   │   ├── mapData.ts              # Overture buildings + OSM geometry loader
│   │   └── dummy/                  # Dummy data for development
│   │       ├── buses.json
│   │       ├── tubes.json
│   │       ├── trains.json
│   │       ├── planes.json
│   │       ├── ships.json
│   │       └── riverboats.json
│   │
│   ├── components/
│   │   ├── Scene.tsx               # R3F Canvas + controls + bloom
│   │   ├── MapTiles.tsx            # London buildings (instanced)
│   │   ├── Water.tsx               # Thames water surface
│   │   ├── TransportLayer.tsx      # Parent transport renderer
│   │   ├── TubeLines.tsx           # Tube lines + animated dots
│   │   ├── BusMarkers.tsx          # Bus capsules on road geometry
│   │   ├── TrainMarkers.tsx        # Train dots on rail geometry
│   │   ├── ShipMarkers.tsx         # Ship capsules on water
│   │   ├── PlaneMarkers.tsx        # Plane capsules in air
│   │   ├── RiverBoats.tsx          # River boat dots on waterway
│   │   ├── StreetLamps.tsx         # Traffic signal markers
│   │   ├── Tooltip.tsx             # Vehicle info tooltip overlay
│   │   ├── JamcamOverlay.tsx       # TFL jamcam image overlay
│   │   ├── LoadingScreen.tsx       # Green neon loading screen
│   │   └── LayerToggles.tsx        # Transport layer toggle buttons
│   │
│   ├── hooks/
│   │   ├── useDataPolling.ts       # Generic polling hook with rate limiting
│   │   ├── useVehiclePosition.ts   # Position inference hook
│   │   └── useCamera.ts            # Camera presets + controls
│   │
│   └── utils/
│       ├── geo.ts                  # Lat/lng ↔ 3D coordinates (Web Mercator)
│       ├── routeInterpolation.ts   # Position along route geometry
│       ├── constants.ts            # London bounds, TfL colors, etc.
│       └── cache.ts                # In-memory cache with TTL
│
├── server/
│   ├── index.ts                    # Express server (proxy + dummy data)
│   ├── proxy.ts                    # API proxy routes
│   ├── dummy.ts                    # Dummy data endpoints
│   └── cache.ts                    # Server-side cache
│
├── public/
│   ├── map-data/                   # Pre-processed GeoJSON (buildings, geometry)
│   ├── .env.example                # API key template
│   └── favicon.ico
│
├── scripts/
│   ├── download-map-data.ts        # Download Overture tiles + convert to GeoJSON
│   └── fetch-osm-geometry.ts       # Fetch Overpass geometry at build time
│
├── docs/
│   ├── prd.md                      # This file
│   ├── plan-architecture.md        # Architecture plan
│   └── api-keys.md                 # API registration links
│
├── vite.config.ts
├── tsconfig.json
├── package.json
├── .gitignore
└── .env.example
```

---

## 4. UI/UX Specifications

### 4.1 Loading Screen

Shown on app launch, dismissed once all data is loaded.

**Visual Design:**
- Dark background (#0a0a0a)
- Green neon (#00ff88) text and progress bars
- Skyline logo (SVG outline of London landmarks)
- Monospace font for all text

**Layout:**
```
        [neon skyline SVG logo]
              ZONE ONE

  A LIVE MAP OF CENTRAL LONDON · EVERY TUBE,
  BUS, TRAIN, RIVERBOAT AND AIRCRAFT MOVING
  IN REAL TIME

  CORE           ████████████████ 100%
  BUILDINGS      ████████████████ 100%
  ROADS          ████████████████ 100%
  LANDUSE        ████████████████ 100%
  EXTRUDE        ████████████░░░░  72%

                     94%
```

**Progress Bars:**
- CORE: Map data loaded
- BUILDINGS: Overture buildings loaded
- ROADS: OSM route geometry loaded
- LANDUSE: Land use data loaded
- EXTRUDE: 3D building extrusion progress
- Percentage shown per bar + overall percentage

**Dismissal:** Screen fades out over 0.5s when all data is loaded.

### 4.2 3D Scene

**Camera Controls:**
- **Left drag** — Orbit around center point
- **Scroll wheel** — Zoom in/out (min: 50m, max: 10,000m)
- **Right drag** — Pan
- **Double-click** — Focus on clicked vehicle

**Camera Presets:**
- **Default**: Angled view of central London (51.5074, -0.1278), 2000m altitude, 45° tilt
- **Reset View**: Returns to default position
- **Top-down**: Orthographic-style view from directly above

**UI Buttons (top-right corner):**
- **ORBIT** — Toggle auto-rotation (~1°/sec)
- **RESET VIEW** — Return to default camera position
- **LAYER TOGGLES** — Small color-coded buttons per transport type

### 4.3 Transport Layer Toggles

Small icon buttons in top-right corner, one per transport type:

| Layer | Button Color | State |
|-------|-------------|-------|
| Tube | TfL line colors (multi) | Toggle on/off |
| Bus | Red (#ff0000) | Toggle on/off |
| Train | Yellow (#ffcc00) | Toggle on/off |
| Plane | White (#ffffff) | Toggle on/off |
| Ship | Teal (#00bcd4) | Toggle on/off |
| River | Green (#00c853) | Toggle on/off |

When toggled off, the layer and all its vehicles are hidden.

### 4.4 Tooltips

**Bus Tooltip** (on hover):
```
┌─────────────────────────────────────┐
│ BUS N343 · SN18KNB                  │  ← Route + Registration (red)
│ TO TRAFALGAR SQUARE, FLEET STREET   │  ← Destinations
│ NEXT WATERLOO BRIDGE / SOUTH BANK   │  ← Next stops
│ 1M 35S                              │  ← ETA (red)
└─────────────────────────────────────┘
```

**Tube Tooltip** (on hover):
```
┌─────────────────────────────────────┐
│ Central Line · Hainault             │  ← Line + Destination
│ NEXT: Leyton · 3 min                │  ← Next station + ETA
│ DELAY: +2 min                       │  ← Delay (if any)
└─────────────────────────────────────┘
```

**Train Tooltip** (on hover):
```
┌─────────────────────────────────────┐
│ Southern · Brighton                 │  ← Operator + Destination
│ NEXT: East Croydon · 5 min          │  ← Next station + ETA
│ ON TIME                             │  ← Status
└─────────────────────────────────────┘
```

**Plane Tooltip** (on hover):
```
┌─────────────────────────────────────┐
│ BAW123 · British Airways            │  ← Callsign + Airline
│ LHR → JFK                           │  ← Origin → Destination
│ ALT: 35,000 FT · SPD: 480 KTS       │  ← Altitude + Speed
│ HDG: 270°                           │  ← Heading
└─────────────────────────────────────┘
```

**Ship Tooltip** (on hover):
```
┌─────────────────────────────────────┐
│ ALMERE 4                            │  ← Vessel name (teal)
│ SPD 0.0 KN · HDG 0°                 │  ← Speed + Heading
│ MMSI 232005500                      │  ← MMSI number
│ Port tender                         │  ← Vessel type
│ 28m × 4m · 1.0m draught             │  ← Dimensions
│ FLAG United Kingdom (UK)            │  ← Flag
└─────────────────────────────────────┘
```

**Design:**
- Dark background (#1a1a1a, 90% opacity)
- Colored left border (matches transport type)
- Monospace font
- Appears near the hovered vehicle
- Disappears after 3s or on click away

### 4.5 Jamcam Overlay

Shown when hovering over a traffic signal (street lamp).

**Layout:**
```
┌─────────────────────────────────────┐
│ SOUTH - TWDS WESTMINSTER            │  ← Camera name (green)
│ Thu 18 Jun 01:38                    │  ← Timestamp
│                                     │
│  [TFL Jamcam Image/Video]           │  ← Live camera feed
│                                     │
│  Gnor Gdns/Buckingham Palace Rd     │  ← Location
│  TFL JAMCAM · LIVE STILL · REFRESHES 12S │ ← Status
└─────────────────────────────────────┘
```

**Data:**
- Image URL: from TfL Jamcam API (`imageUrl` field)
- Video URL: from TfL Jamcam API (`videoUrl` field)
- Camera position: from TfL Jamcam API (`lat`, `lon`)
- Matched to nearest street lamp by proximity

---

## 5. Visual Design

### 5.1 Color Palette

| Element | Color | Hex |
|---------|-------|-----|
| Background | Dark | #0a0a0a |
| Buildings | Dark grey | #1a1a1a |
| Roads | Medium grey | #2a2a2a |
| Water | Dark blue | #0a1628 |
| Neon glow | Green | #00ff88 |
| Tube - Bakerloo | Brown | #cc9966 |
| Tube - Central | Red | #ee2024 |
| Tube - Circle | Yellow | #ffcd00 |
| Tube - District | Green | #007828 |
| Tube - Hammersmith & City | Pink | #f4a460 |
| Tube - Jubilee | Silver | #979797 |
| Tube - Metropolitan | Purple | #8f147d |
| Tube - Northern | Black | #000000 |
| Tube - Piccadilly | Dark blue | #001689 |
| Tube - Victoria | Blue | #0099cc |
| Tube - Waterloo & City | Turquoise | #60ac57 |
| Bus | Red | #ff0000 |
| Train | Yellow | #ffcc00 |
| Plane | White | #ffffff |
| Ship | Teal | #00bcd4 |
| River boat | Green | #00c853 |
| Street lamp | Yellow | #ffcc00 |

### 5.2 Lighting

- **Ambient light**: dim, cool white (intensity: 0.3)
- **Directional light**: warm, from above (intensity: 0.5)
- **Bloom post-processing**: UnrealBloomPass (strength: 1.5, radius: 0.4, threshold: 0.6)
- **No shadow casting** on buildings (performance)
- **Shadow casting** on vehicles only

### 5.3 Vehicle Models

| Transport | Shape | Color | Size |
|-----------|-------|-------|------|
| Tube train | Glowing dot (sphere) | Line color | 0.8m diameter |
| Bus | Capsule/box | Red (#ff0000) | 10m × 2.5m × 3m |
| Train | Glowing dot (sphere) | Yellow (#ffcc00) | 0.8m diameter |
| Plane | Capsule/box | White (#ffffff) | 15m length |
| Ship | Capsule/box | Teal (#00bcd4) | 28m × 4m |
| River boat | Glowing dot (sphere) | Green (#00c853) | 0.8m diameter |
| Street lamp | Small glowing box | Yellow (#ffcc00) | 0.5m × 0.5m |

### 5.4 Neon Lines

Transport routes rendered as glowing lines:
- **Tube lines**: Each line gets its TfL color, rendered as a continuous path
- **Bus routes**: Red lines for all bus routes
- **Rail lines**: Yellow lines for National Rail
- **Waterways**: Blue lines for the Thames
- **Roads**: Grey lines for major roads

All lines use emissive materials + bloom post-processing for the neon glow effect.

---

## 6. Animation & Inference Engine

### 6.1 Animation Framework

- **Frame rate**: 60fps via `requestAnimationFrame`
- **Interpolation**: Linear interpolation between known positions
- **Path following**: Vehicles snap to nearest point on road/rail geometry
- **No flying**: Vehicles stay on roads/tracks at correct elevation

### 6.2 Tube Train Animation

**Data source**: TfL Metro Live Arrivals API
**Polling interval**: 30s
**Animation**:

1. Fetch arrivals for all Tube stations
2. For each arriving train, find its line geometry from OSM
3. Calculate position on line: `position = (eta / avg_headway) * line_length`
4. Place train dot at calculated position on the line
5. Each frame, move train dot closer to destination based on time elapsed
6. When train arrives at station, remove it and spawn a new one at origin

**Visual**: Glowing dot on the colored tube line, moving along the track.

### 6.3 Bus Animation

**Data sources**: TfL Bus RealTime API (GPS buses) + TfL Live Arrivals (inferred buses)
**Polling interval**: 30s
**Animation**:

1. For GPS buses: use live lat/lng, snap to nearest road segment
2. For inferred buses: use arrival data, interpolate along bus route geometry
3. Calculate heading: direction to next point on road geometry
4. Orient bus capsule to face forward along the road
5. Each frame, move bus along the road geometry toward next stop

**Visual**: Red capsule on the road, facing forward, moving toward destination.

### 6.4 Train Animation

**Data source**: Darwin DBRS Departure Boards API
**Polling interval**: 30s
**Animation**:

1. Fetch departure boards for all National Rail stations in London
2. For each arriving train, find its rail geometry from OSM
3. Calculate position: `position = (eta / avg_journey_time) * segment_length`
4. Place train dot at calculated position on the rail line
5. Each frame, move train dot closer to destination based on time elapsed
6. When train arrives, remove and spawn new one at origin

**Visual**: Glowing yellow dot on the rail line, moving along the track.

### 6.5 Plane Animation

**Data source**: ADSBx v2 API
**Polling interval**: 10s
**Animation**:

1. Fetch all aircraft in London bounding box
2. Position: lat/lng projected to X/Z, altitude converted to Y
3. Orientation: rotate to match `true_track` (0°=north, 90°=east)
4. Pitch: slight tilt based on `vert_rate` (positive = climbing)
5. Each frame, move plane by `velocity * dt` in the heading direction

**Visual**: White capsule in the air, oriented by heading, at correct altitude.

### 6.6 Ship Animation

**Data source**: AIS Hub API
**Polling interval**: 60s
**Animation**:

1. Fetch all vessels in Thames bounding box
2. Position: lat/lng projected to X/Z, Y = -0.5 (in river)
3. Orientation: rotate to match `heading`
4. Each frame, move ship by `SOG * dt` in the heading direction

**Visual**: Teal capsule on the water, oriented by heading.

### 6.7 River Boat Animation

**Data source**: TfL River API + Live Arrivals
**Polling interval**: 30s
**Animation**: Same as tube trains but on waterway geometry

**Visual**: Glowing green dot on the Thames, moving between river bus stops.

---

## 7. Performance Specifications

### 7.1 Draw Call Budget

| Element | Count | Draw Calls |
|---------|-------|------------|
| Buildings (merged by height) | ~4 groups | 4 |
| Tube lines | ~13 lines | 13 |
| Bus routes | ~100 routes | 100 |
| Rail lines | ~20 lines | 20 |
| Waterway | 1 | 1 |
| Roads | ~50 | 50 |
| Street lamps | ~500 | 1 (instanced) |
| Tube trains | ~200 | 1 (instanced) |
| Buses | ~800 | 1 (instanced) |
| Trains | ~100 | 1 (instanced) |
| Planes | ~50 | 1 (instanced) |
| Ships | ~20 | 1 (instanced) |
| River boats | ~10 | 1 (instanced) |
| **Total** | | **~212** |

Target: **< 250 draw calls** for 60fps on mid-tier hardware.

### 7.2 Memory Budget

| Element | Size |
|---------|------|
| Overture buildings (GeoJSON) | ~200MB |
| OSM geometry (GeoJSON) | ~50MB |
| Three.js library | ~15MB |
| React + dependencies | ~5MB |
| App code | ~2MB |
| **Total** | **~272MB** |

Target: **< 300MB** total memory usage.

### 7.3 Optimization Strategies

1. **Instanced meshes** — One draw call per transport type (not per vehicle)
2. **Building merging** — Group buildings by height tier, merge into 4 instanced meshes
3. **Frustum culling** — Only render elements in camera view + margin
4. **LOD (future)** — Simplify geometry at distance (planned enhancement)
5. **Pixel ratio cap** — Max 2x device pixel ratio
6. **Geometry simplification** — Buildings as simple boxes (no detail)
7. **No building shadows** — Shadows only on vehicles
8. **Single bloom pass** — One post-processing effect for all glow

### 7.4 Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| Frame rate | 60fps | On M1/M2 or equivalent |
| Minimum frame rate | 30fps | On integrated graphics |
| Initial load time | < 10s | From page load to interactive |
| Memory usage | < 300MB | Total, including Three.js |
| Draw calls | < 250 | During normal operation |
| API latency | < 2s | Per data source |

---

## 8. Backend API Specification

### 8.1 Proxy Server Endpoints

The Express proxy server provides clean endpoints to the frontend. The frontend never calls external APIs directly.

#### Tube Data
```
GET /api/tube/arrivals
Response: {
  "stations": [{
    "id": "940GZZLUAC",
    "name": "London Bridge",
    "lat": 51.5049,
    "lon": -0.0863,
    "arrivals": [{
      "lineName": "Jubilee",
      "destination": "Stanmore",
      "minutesToStation": 3,
      "lineId": 7
    }]
  }]
}
```
- TTL: 30s
- Cache: in-memory Map

#### Bus Data
```
GET /api/buses
Response: {
  "buses": [{
    "id": "9100GAT8901",
    "route": "N343",
    "registration": "SN18KNB",
    "destination": "Trafalgar Square",
    "nextStops": ["Waterloo Bridge", "South Bank"],
    "eta": 95,
    "lat": 51.5074,
    "lon": -0.1278,
    "heading": 180,
    "type": "gps" | "inferred"
  }]
}
```
- TTL: 30s
- Cache: in-memory Map

#### Train Data
```
GET /api/trains
Response: {
  "trains": [{
    "id": "9100GLVC_001",
    "operator": "Southern",
    "destination": "Brighton",
    "nextStation": "East Croydon",
    "eta": 300,
    "lat": 51.4890,
    "lon": -0.1440,
    "heading": 180,
    "status": "on_time" | "delayed"
  }]
}
```
- TTL: 30s
- Cache: in-memory Map

#### Plane Data
```
GET /api/planes
Response: {
  "planes": [{
    "icao24": "400b21",
    "callsign": "BAW123",
    "airline": "British Airways",
    "origin": "LHR",
    "destination": "JFK",
    "lat": 51.5098,
    "lon": -0.1234,
    "altitude": 3500,
    "speed": 175,
    "heading": 268,
    "vertRate": -200
  }]
}
```
- TTL: 10s
- Cache: in-memory Map

#### Ship Data
```
GET /api/ships
Response: {
  "ships": [{
    "mmsi": 235006759,
    "name": "ALMERE 4",
    "type": "Port tender",
    "lat": 51.4900,
    "lon": -0.1200,
    "speed": 5,
    "heading": 42,
    "length": 28,
    "width": 4,
    "draught": 1.0,
    "flag": "United Kingdom",
    "destination": "Tower Bridge"
  }]
}
```
- TTL: 60s
- Cache: in-memory Map

#### Jamcam Data
```
GET /api/jamcams
Response: {
  "jamcams": [{
    "id": "JamCams_00001.07450",
    "name": "Piccadilly Circus",
    "lat": 51.5096,
    "lon": -0.13484,
    "available": true,
    "imageUrl": "https://s3-eu-west-1.amazonaws.com/jamcams.tfl.gov.uk/00001.07450.jpg",
    "videoUrl": "https://s3-eu-west-1.amazonaws.com/jamcams.tfl.gov.uk/00001.07450.mp4",
    "view": "Piccadilly Circus"
  }]
}
```
- TTL: 60s
- Cache: in-memory Map

#### Map Data
```
GET /api/map/buildings
Response: GeoJSON FeatureCollection of building footprints with height

GET /api/map/geometry
Response: GeoJSON FeatureCollection of road/rail/waterway geometry
```
- TTL: persistent (cached indefinitely)
- Source: static files from `public/map-data/`

### 8.2 Dummy Data Format

During development, the proxy serves dummy data that matches the real API response structure. This allows the frontend to be built and tested without API keys.

Dummy data files are in `src/data/dummy/` and served by the Express server at the same endpoints.

### 8.3 Rate Limiting

The proxy server enforces rate limits per data source:

| Source | Limit | Enforcement |
|--------|-------|-------------|
| TfL | 10 req/s | Queue + delay |
| Darwin | 100k credits/day | Track usage |
| ADSBx | 1 req/10s | Strict 10s interval |
| AIS Hub | 1 req/60s | Strict 60s interval |

The proxy uses a token bucket algorithm to ensure we never exceed limits.

---

## 9. Coordinate System & Geometry

### 9.1 Coordinate Mapping

- **Origin**: London center (51.5074°N, -0.1278°W) mapped to (0, 0, 0)
- **Projection**: Web Mercator (EPSG:3857) for lat/lng → X/Z
- **Y-axis**: Altitude/elevation (1 unit = 1 meter)
- **Scale**: 1 Three.js unit = 1 real-world meter

### 9.2 Elevation Layers

| Layer | Y Position | Notes |
|-------|-----------|-------|
| Ground | Y = 0 | Flat dark surface |
| River (Thames) | Y = -0.5 | Below ground level |
| Roads | Y = 0.1 | Slightly above ground |
| Rail lines | Y = 0.2 | Above roads where divergent |
| Buildings | Y = 0 to height | Extruded from ground |
| Ships | Y = -0.5 | In the river |
| Buses/Trains | Y = 0.1 | On road level |
| Planes | Y = altitude | From API data (meters) |

### 9.3 Vehicle Positioning

All vehicles are projected onto the nearest point on their respective geometry:
- Buses → nearest point on road geometry
- Trains → nearest point on rail geometry
- Ships → nearest point on waterway geometry
- Planes → direct lat/lng (no geometry needed)
- River boats → nearest point on waterway geometry

Vehicles never fly over buildings because they are constrained to the geometry path.

---

## 10. Development Phases

### Phase 1: Setup (Done)
- [x] Vite + React + TypeScript scaffold
- [x] Three.js + R3F + dependencies installed
- [x] Project structure created

### Phase 2: Infrastructure
- [ ] Express proxy server with dummy data endpoints
- [ ] Zustand store with types
- [ ] Coordinate conversion utilities (geo.ts)
- [ ] Cache utility with TTL
- [ ] Polling hook with rate limiting
- [ ] `concurrently` setup for dev server

### Phase 3: Loading Screen
- [ ] Green neon loading screen component
- [ ] Progress bars per data layer
- [ ] Skyline SVG logo
- [ ] Fade-out on complete

### Phase 4: Map Foundation
- [ ] Overture buildings download script
- [ ] Parquet → GeoJSON conversion
- [ ] Building instancing (by height tier)
- [ ] Thames water surface
- [ ] Ground plane

### Phase 5: Route Geometry
- [ ] Overpass geometry fetch script
- [ ] Road/rail/waterway rendering
- [ ] Neon line rendering with bloom
- [ ] Street lamp placement

### Phase 6: Tube Lines
- [ ] Tube line geometry from OSM
- [ ] TfL line colors
- [ ] Animated tube dots on lines
- [ ] Dummy tube data

### Phase 7: Bus Markers
- [ ] Bus capsule rendering
- [ ] Road snapping
- [ ] Bus orientation along road
- [ ] Inference engine (position along route)
- [ ] Dummy bus data

### Phase 8: Train Markers
- [ ] Train dot rendering
- [ ] Rail geometry snapping
- [ ] Train orientation along rail
- [ ] Dummy train data

### Phase 9: Plane Markers
- [ ] Plane capsule rendering
- [ ] Heading-based orientation
- [ ] Altitude mapping
- [ ] Dummy plane data

### Phase 10: Ship Markers
- [ ] Ship capsule rendering
- [ ] Heading-based orientation
- [ ] Water-level positioning
- [ ] Dummy ship data

### Phase 11: River Boats
- [ ] River boat dot rendering
- [ ] Waterway geometry snapping
- [ ] Dummy river boat data

### Phase 12: UI Polish
- [ ] Transport tooltips per type
- [ ] Layer toggle buttons
- [ ] Reset view button
- [ ] Orbit toggle
- [ ] Camera presets

### Phase 13: Jamcam Overlay
- [ ] TfL Jamcam API integration
- [ ] Street lamp to jamcam matching
- [ ] Jamcam overlay component
- [ ] Hover interaction

### Phase 14: Real API Integration
- [ ] Replace dummy data with real TfL calls
- [ ] Replace dummy data with real Darwin calls
- [ ] Replace dummy data with real ADSBx calls
- [ ] Replace dummy data with real AIS Hub calls
- [ ] API key configuration
- [ ] Error handling + retry logic

### Phase 15: Production
- [ ] Vite production build
- [ ] Docker setup (optional)
- [ ] Deployment configuration
- [ ] Performance testing

---

## 11. API Key Registration Checklist

Before running the app, register for these accounts:

- [ ] **TfL API Key** — https://api.tfl.gov.uk/ (free)
- [ ] **ADSBx API Key** — https://www.adsbexchange.com/products/enterprise-api/ (free community tier)
- [ ] **AIS Hub Username** — https://www.aishub.net/join-us (free)
- [ ] **Darwin DBRS Account** — https://realtime.nationalrail.co.uk/DBRSLogin/ (free)

Store keys in `.env.local` (gitignored). Template in `.env.example`.

---

## 12. Future Enhancements (Out of Scope for V1)

- Performance settings menu (low/medium/high quality)
- Mobile support (touch controls, responsive layout)
- Keyboard shortcuts (R=reset, 1-6=toggle layers, F=focus)
- Accessibility (screen reader support, keyboard navigation)
- LOD (Level of Detail) for buildings at distance
- Real-time weather overlay
- Historical data playback
- Multi-city support (Paris, New York, Tokyo)
- Export/screenshot functionality
- Social sharing

---

## 13. Scoring Summary

| Criteria | Score | Notes |
|----------|-------|-------|
| Feasibility | 9/10 | All APIs are real and accessible |
| Data freshness | 9/10 | All sources are real-time or near-real-time |
| Visual fidelity | 9/10 | R3F + bloom + instanced meshes = excellent |
| Complexity | 7/10 | 6 data sources + inference engine = significant |
| Performance | 8/10 | Instanced meshes + building merging = good |
| Scope clarity | 10/10 | Fully specified, no ambiguity |

**Overall: 8.8/10** — solid, well-scoped project with clear implementation path.
