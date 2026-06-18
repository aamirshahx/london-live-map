# London Live 3D Map - Architecture Plan

## Overview
A real-time 3D map of London showing live transport: Tube, bus, riverboat,
National Rail, planes, and ships. Transport without GPS (buses, trains) is
inferred from arrival countdowns and animated along route/track geometry.

## Data Sources

| Source | Transport | Endpoint | Rate Limit | Auth |
|--------|-----------|----------|------------|------|
| TfL API | Tube, Bus, River | api.tfl.gov.uk | 10 req/s, 100k/day | Free key |
| Darwin DBRS | National Rail | webapi.datatools.stack.com | 100k credits | Free account |
| OpenSky | Planes | opensky-network.org | 1 req/10s | Free |
| AIS Hub | Ships | aisHub.net | 1 req/30s | Free |
| Overture Maps | Map tiles | docs.overturemaps.org | - | Free |
| Overpass API | Road/rail geometry | overpass-api.de | - | Free |

## Architecture

```
+-----------------------------------------------------+
|                    React App                          |
|  +-------------+  +----------+  +-----------------+  |
|  |  UI Layer   |  |  Map UI  |  | Transport Panel |  |
|  |  (HTML/CSS) |  | (R3F)    |  |  (Tooltips)     |  |
|  +------+------+  +----+-----+  +--------+--------+  |
|         |               |                    |        |
|  +------v---------------v--------------------v-------+|
|  |              Zustand Store (state)                ||
|  |  - mapData, transportVehicles, selectedVehicle    ||
|  +------+-------------------------------------------+|
|         |                                          |
|  +------v---------------+           +--------------v------+|
|  |  Data Fetchers        |           |  Inference Engine  ||
|  |  (polling)            |           |  (no-GPS pos)      ||
|  +-----------------------+           +---------------------+|
+-------------------------------------------------------------+
```

## Directory Structure

```
london-live-map/
+-- src/
|   +-- main.tsx
|   +-- App.tsx
|   +-- index.css
|   +-- App.css
|   +-- store/
|   |   +-- index.ts           # Zustand store
|   |   +-- types.ts           # Shared types
|   |   +-- selectors.ts       # Derived state
|   +-- data/
|   |   +-- fetchers/
|   |   |   +-- tfl.ts         # TfL API client
|   |   |   +-- darwin.ts      # Darwin DBRS client
|   |   |   +-- opensky.ts     # OpenSky client
|   |   |   +-- ais.ts         # AIS Hub client
|   |   +-- inference.ts       # Position inference engine
|   |   +-- mapData.ts         # Overture/OSM tile loader
|   +-- components/
|   |   +-- Scene.tsx          # R3F Canvas + controls
|   |   +-- MapTiles.tsx       # London terrain/buildings
|   |   +-- TransportLayer.tsx # Parent transport renderer
|   |   +-- TubeLines.tsx      # Tube line geometries
|   |   +-- BusMarkers.tsx     # Bus 3D markers
|   |   +-- TrainMarkers.tsx   # Train 3D markers
|   |   +-- ShipMarkers.tsx    # Ship 3D markers
|   |   +-- PlaneMarkers.tsx   # Plane 3D markers
|   |   +-- RiverBoats.tsx     # River bus markers
|   |   +-- Tooltip.tsx        # Vehicle info tooltip
|   +-- hooks/
|   |   +-- useDataPolling.ts  # Generic polling hook
|   |   +-- useVehiclePosition.ts # Inference hook
|   +-- utils/
|       +-- geo.ts             # Lat/lng <-> 3D coords
|       +-- routeInterpolation.ts # Position along route
|       +-- constants.ts       # London bounds, colors
+-- public/
|   +-- .env.local             # API keys (gitignored)
+-- docs/
|   +-- plan-architecture.md
|   +-- api-keys.md
+-- vite.config.ts
+-- tsconfig.json
+-- package.json
```

## Key Design Decisions

1. Zustand for state - lightweight, works well with R3F
2. Polling over WebSockets - simpler, matches API rate limits
3. Position inference - linear interpolation along known route geometry
4. Overture Maps for base tiles - 3D building footprints
5. Overpass for route geometry - rail lines, bus routes, river paths
6. Three.js instanced meshes - performant rendering of many vehicles

## Inference Engine Logic

For buses without live GPS:
1. Fetch TfL stop arrivals: GET /Metro/Live/Arrivals/{stationId}
2. Get route geometry from OSM/Overpass
3. Interpolate position along route based on time-to-stop
4. Animate vehicle along path each frame

For trains without GPS:
1. Fetch Darwin departure boards for nearby stations
2. Get rail track geometry from OSM
3. Interpolate position between stations based on scheduled ETA
4. Animate along rail geometry

## Scoring

| Criteria | Score | Notes |
|----------|-------|-------|
| Feasibility | 7/10 | OpenSky rate limit (1/10s) is tight for full London |
| Data freshness | 8/10 | TfL/Darwin are real-time, AIS/ADS-B are near-real-time |
| Visual fidelity | 9/10 | R3F + instanced meshes = smooth at scale |
| Complexity | 6/10 | 6 data sources + inference engine = significant scope |
| Performance | 7/10 | Instanced meshes help, but 6 fetchers = CPU cost |

Overall: 7.4/10 - solid concept, rate limits are the main risk.

## Development Pattern
TDD recommended for the inference engine (hardest logic).
Rest is integration-heavy - implement-and-test works better.
