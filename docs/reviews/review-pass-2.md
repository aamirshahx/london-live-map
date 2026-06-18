# London Live 3D Map — Implementation Readiness Review (Pass 2)

**Date:** 2026-06-18
**Reviewer:** Staff Software Engineer
**Scope:** Pre-implementation architecture review
**Documents reviewed:** `prd.md`, `architecture.md`, `plan-architecture.md`, `alignment.md`, `review-pass-1.md`
**Codebase state:** Scaffold only (Vite + React + Three.js). No implementation.
**Model** Qwen 3.6 27B


## Executive Summary

**Verdict: Conditional GO.** The project is buildable, but two critical data-delivery problems and seven high-risk gaps must be resolved before Phase 2 starts. The core concept and tech stack are sound. The documents contain optimistic assumptions about browser memory capacity, API rate limits, and deployment that will fail at scale.

### Severity Breakdown

| Severity | Count | Theme |
|----------|-------|-------|
| Critical | 2 | Browser crash on data load, API rate limit exhaustion |
| High | 7 | State thrashing, VRAM overflow, missing deployment, Darwin credits, adapter pattern, late API integration, build script blocker |
| Medium | 14 | Missing deps, interpolation, inference location, LOD, bloom cost, raycasting, cache eviction, retries, module boundaries, testing, deduplication, secrets, CORS, logging, attribution |
| Low | 5 | TS version, state split, shared types, bounding box filter, timezone handling |

### Must Fix Before Implementation

1. **Replace ADR-004** — static GeoJSON will crash the browser. Use Draco-compressed glTF.
2. **Redesign the backend** from proxy to aggregator with push updates. The proxy model fails with 2+ users.
3. **Pick a deployment platform** now, not in Phase 15. The choice affects backend structure.

---

## 1. Implementation Feasibility

### 1.1 Technology Choices

#### Medium — Missing dependencies in package.json

`package.json` only lists Three.js, R3F, Drei, React, and ReactDOM. The architecture references Zustand, Express, `@react-three/postprocessing`, `d3-geo`, `@turf/turf`, and `concurrently`, none of which are in the dependency tree. This will be discovered mid-build when imports fail.

**Recommended change:** Add a dependency table to `architecture.md` listing every package, version constraint, and purpose. Include both frontend and backend dependencies so the implementer has a complete picture before starting.

**Estimated impact:** 1 hour of documentation.

#### Low — TypeScript 6.x is cutting-edge

`typescript: "~6.0.2"` is the latest major. The Three.js type ecosystem (`@types/three@0.184.1`) and R3F/Drei may not have been tested against TS 6. Type conflicts in `@types/three` are common and can consume hours chasing library bugs.

**Recommended change:** Pin `typescript: "~5.8.2"` until the R3F/Drei stack explicitly supports TS 6. Upgrade later when the ecosystem catches up.

**Estimated impact:** 5 minutes to downgrade.

---

### 1.2 Frontend Architecture

#### Critical — 400–800MB GeoJSON will crash the browser

`ADR-004` states: "Download and convert Parquet → GeoJSON at build time. Serve static GeoJSON files." The PRD acknowledges 400–800MB for Greater London buildings. Loading a 500MB JSON payload into a browser tab will:

- Freeze the main thread for 10–30 seconds during `JSON.parse`
- Consume 1–2GB of heap memory (JSON → JS object expansion is 2–4× the raw size)
- Crash on any device under 8GB RAM
- Block the loading screen, making it appear frozen

This is not a performance degradation. It is a hard failure where the tab becomes unresponsive and the browser offers to kill it.

**Recommended change (pick one):**

1. **Merge + quantize in the build step (recommended).** Write a build script that converts Overture Parquet → single merged glTF with Draco compression. Target <50MB compressed. Load in Three.js via `useGLTF` from Drei. This keeps the stack simple and avoids adding Cesium.
2. **3D Tiles (Cesium ion or i3dm/gltf).** Stream buildings by frustum + LOD. Best option for scale but adds Cesium/ion dependency and significant architectural complexity.
3. **Tile GeoJSON manually.** Partition by zone, lazy-load by camera position. More work, less tooling support.

Option 1 is the minimum viable fix. Rewrite ADR-004 to reflect Draco-compressed glTF instead of static GeoJSON.

**Estimated impact:** 2–3 days of build-script work. Requires rewriting ADR-004.

#### High — Zustand store will thrash with thousands of vehicle updates

The architecture stores all vehicle positions in a single Zustand store. With ~800 buses, ~200 trains, ~50 planes, and ~30 ships updating every 10–30 seconds, every update triggers:

- Zustand state replacement (shallow comparison on the entire state tree)
- React re-render of every subscribing component
- R3F reconciliation of every vehicle marker component

Even with `useShallow`, updating 1000+ objects through React state at 60fps animation is the wrong pattern. React's reconciliation cycle is not designed for high-frequency position updates.

**Recommended change:** Use mutable refs for vehicle positions. Store positions in a `Map<string, Vector3>` held in a `useRef`. Update directly in the `useFrame` loop. Zustand only holds metadata: selected vehicle, layer toggles, tooltip state, and loading progress. The animation loop reads the ref, not React state.

**Estimated impact:** Architectural change to the store design. ~1 day to restructure.

#### Medium — No client-side interpolation between polling updates

Vehicles are polled every 10–30 seconds but the animation loop runs at 60fps. Without interpolation, vehicles will "jump" between positions every poll cycle. The PRD says "animated along actual road/rail geometry" but does not specify the interpolation algorithm.

**Recommended change:** Implement lerp-based interpolation in the `useFrame` loop. Store `previousPosition`, `currentPosition`, and `arrivalTime` per vehicle. Animate between them with `t = (now - arrivalTime) / pollInterval`. Clamp `t` to [0, 1].

**Estimated impact:** ~4 hours to implement across all vehicle types.

---

### 1.3 Backend Architecture

#### Critical — Express proxy will exhaust API rate limits with 2+ concurrent users

`ADR-006` describes a proxy that forwards requests. TfL allows 10 req/s and 100,000 calls/day. Each user polling cycle triggers approximately:

- Tube: ~50 station requests per 30s poll
- Bus: batched but still multiple calls
- Darwin: per-station calls
- ADSBx: 1 call per 10s
- AIS: 1 call per 60s

One user polling = ~2–3 req/s average. Two simultaneous users = rate limit breach. The free API tier will return 429s, and repeated abuse will result in key suspension.

**Recommended change:** The backend must be an aggregator, not a proxy:

1. Poll each API on a fixed schedule owned by the server, not triggered by client requests
2. Cache results in memory with TTL
3. Serve cached data to all connected clients
4. Use Server-Sent Events (SSE) or long-polling to push updates to clients instead of clients pulling

This is a fundamental architectural change from ADR-006. The backend is not a reverse proxy — it is a data aggregation service with a simple HTTP/SSE API.

**Estimated impact:** 2–3 days to redesign the backend. Requires SSE or polling redesign.

#### High — Darwin DBRS credit model is not accounted for

Darwin uses a credit system (100,000 credits/day). The PRD does not specify how many credits each call costs. A departure board call may cost 1–10 credits depending on scope. Polling 50+ stations every 30 seconds could burn through 100,000 credits in hours.

**Recommended change:**
1. Research Darwin credit cost per endpoint call before implementation
2. Add credit tracking to the backend with a daily budget alert
3. Reduce polling frequency or station count if credits are tight
4. Consider polling only major stations (Victoria, Paddington, King's Cross, Liverpool Street, Waterloo, Euston, Marylebone, Charing Cross, Cannon Street, Blackfriars) instead of all stations

**Estimated impact:** 1 hour research + potential scope reduction.

#### High — No deployment strategy

Phase 15 says "Docker setup (optional)" and "Deployment configuration" with zero detail. The frontend (Vite static files) and backend (Express long-running process) have fundamentally different deployment requirements. Static hosting platforms (Vercel, Netlify, Cloudflare Pages) kill the backend process. You need a platform that runs both, or split them into two services.

**Recommended change:** Pick a platform before Phase 2:

1. **Fly.io** — runs both frontend proxy and backend in one app. Persistent processes. Cheap for this scale.
2. **Render** — static frontend + web service backend. Simple setup.
3. **Railway** — similar to Render, good for Node.js services.

The choice affects how the backend is structured (SSE requires persistent connections, which some platforms terminate after timeouts).

**Estimated impact:** 1 hour decision + 2 days setup.

#### Medium — No API error handling or graceful degradation strategy

The architecture mentions "error handling + retry logic" in Phase 14 but provides no strategy. Scenarios not covered:

- TfL returns 503 for 5 minutes
- ADSBx key is invalid or revoked
- Darwin returns empty results
- Overpass times out on geometry fetch
- Network partition between backend and external API

The frontend has no way to distinguish "no data" from "API down."

**Recommended change:** Define per-source error handling:

- **Circuit breaker:** after N consecutive failures, stop polling that source and mark it as "offline" in the UI
- **Retry with exponential backoff:** 1s → 2s → 4s → 8s → cap at 30s
- **Stale data:** serve last-known-good data with a "stale" flag
- **UI indicator:** show a per-layer status badge (green/yellow/red) in the transport panel

**Estimated impact:** ~6 hours to implement circuit breaker + status UI.

#### Medium — Inference engine location is ambiguous

The PRD mentions "Inference Engine" but does not specify whether it runs on the backend or frontend. `plan-architecture.md` places it in the frontend. `review-pass-1.md` recommends backend. The architecture document does not resolve this.

**Why it matters:**
- Frontend inference: every client recalculates positions. Wastes CPU, inconsistent across clients.
- Backend inference: single source of truth, but requires the server to hold OSM geometry in memory.

**Recommended change:** Put inference on the backend. The server already holds OSM geometry (it's static). It calculates `(x, y, z)` for each vehicle and sends coordinates to the frontend. The frontend only renders. This is cleaner, more consistent, and reduces frontend CPU load.

**Estimated impact:** ~4 hours to move inference logic to backend.

---

### 1.4 Data Flow

#### Medium — No adapter pattern for API response normalization

Each API returns a different schema. The frontend will be tightly coupled to TfL/Darwin/ADSBx/AIS response shapes. When any API changes a field name, the frontend breaks. TfL changed their API in 2024 and broke dozens of integrations.

**Recommended change:** Implement an adapter layer in the backend:

```
External API → Adapter → Unified Transport Schema → Frontend
```

Unified schema:

```typescript
interface Vehicle {
  id: string;
  type: 'bus' | 'tube' | 'train' | 'plane' | 'ship' | 'riverBoat';
  position: { lat: number; lng: number; alt?: number };
  heading?: number;
  speed?: number;
  metadata: Record<string, unknown>;
  timestamp: number;
  confidence: 'gps' | 'inferred';
}
```

**Estimated impact:** ~4 hours to define schema + adapters.

#### Medium — Coordinate system inconsistencies across APIs

All sources use WGS84 (good), but the conversion to Web Mercator (EPSG:3857) and then to Three.js world space must be consistent. A 0.001° error in conversion equals ~100m offset at London latitude. Vehicles will appear in the wrong place.

**Recommended change:**
1. Use `d3-geo` or `proj4` for coordinate transforms. Do not implement the math manually.
2. Write tests that verify known coordinates: Big Ben (51.5007, -0.1246), Heathrow (51.4700, -0.4543), Greenwich (51.4826, -0.0077).
3. Document the transform pipeline: `WGS84 → WebMercator → Three.js World`.

**Estimated impact:** ~2 hours (library usage + tests).

#### Medium — TfL arrival data returns duplicate trains

TfL's `/Metro/Live/Arrivals` endpoint can return the same train at multiple stations. It is an arrival board, not a unique vehicle list. Without deduplication, the same train appears 3–5 times on the map.

**Recommended change:** The inference engine must deduplicate. Strategy: group arrivals by `lineName` + `destination` + similar `minutesToStation`, then place one train per group. Document this explicitly in the inference spec.

**Estimated impact:** ~3 hours in the inference engine.

---

### 1.5 State Management

#### Low — No distinction between "animation state" and "data state"

The Zustand store conflates two concerns:
1. **Data state:** vehicle positions, layer toggles, selected vehicle
2. **Animation state:** interpolation targets, camera transitions, loading progress

Mixing them causes unnecessary re-renders when animation state changes.

**Recommended change:** Split into two stores: `useDataStore` (Zustand, for data) and a ref-based animation system for `useFrame` interpolation.

**Estimated impact:** ~2 hours refactor.

---

## 2. Three.js / Graphics Review

### 2.1 GPU Performance & Draw Calls

#### High — Building count will exceed practical InstancedMesh limits

Greater London has ~1.5M+ building footprints. Even with merging, a single `InstancedMesh` with 1M+ instances will:

- Consume ~12MB for the instance matrix buffer alone
- Require per-instance color updates that touch the full buffer every frame
- Cause GPU driver stalls on integrated graphics

**Recommended change:**
1. Filter buildings: only render buildings > 3m height (removes ~40% of small structures)
2. Merge buildings within 5m into a single box (reduces count by ~60%)
3. Target <100K instances across all tiers
4. Use `mergeBufferGeometries` for static geometry, not InstancedMesh

**Estimated impact:** ~2 days for the build script to implement filtering + merging.

#### High — No LOD system for buildings

The PRD lists LOD as "Future Enhancement (Out of Scope for V1)". For a 30km² city, this is a mistake. Buildings 5km from the camera are 2 pixels wide. Rendering them at full detail wastes GPU cycles.

**Recommended change:** Make LOD a V1 requirement:
1. **Distance culling:** do not render buildings beyond 5km. ~20 lines of code in the render loop.
2. **Height-based simplification:** buildings > 2km away become uniform-height boxes.
3. **Frustum culling:** Three.js does this automatically, but ensure bounding boxes are correct.

**Estimated impact:** ~4 hours to implement distance-based culling.

#### Medium — Bloom post-processing on a 30km scene is expensive

`UnrealBloomPass` renders the full scene at full resolution, then applies a multi-pass blur. At 1920×1080 with bloom, this adds:
- 1 full-screen render pass (emissive extraction)
- 2–4 blur passes (horizontal + vertical, multiple resolutions)
- 1 composite pass

On mid-range GPUs this is ~3–5ms per frame, eating 18–30% of the 16.6ms frame budget.

**Recommended change:**
1. Run bloom at half resolution (`acuity: 0.5` or custom resolution scale)
2. Only bloom emissive objects (use `Selection` pass to isolate neon lines and vehicles)
3. Consider `EffectComposer` with `bypass` on non-emissive frames

**Estimated impact:** ~2 hours to tune bloom settings.

---

### 2.2 Instancing Strategy

#### Medium — Per-instance buffer updates on large meshes

The architecture uses `setColorAt()` for per-instance colors. For vehicles (<5K instances), `needsUpdate = true` is fine. For buildings (potentially 50K+ instances), uploading the full buffer every frame is wasteful.

**Recommended change:**
- Vehicles: `needsUpdate` is fine for <5K instances
- Buildings: never update after initial upload. Use static colors.
- Use `instanceMatrix` updates only for vehicles (position changes), not buildings.

**Estimated impact:** Already implied by the architecture. Needs explicit documentation.

---

### 2.3 Geometry Management

#### Medium — OSM road/rail geometry will be large in memory

Overpass query for Greater London roads + rail + waterways returns ~50–100MB of GeoJSON. Converted to Three.js `LineSegments` or `BufferGeometry`:

- Road network: ~2M vertices × 12 bytes = 24MB
- Rail network: ~500K vertices × 12 bytes = 6MB
- Waterway: ~100K vertices × 12 bytes = 1.2MB

Total geometry: ~31MB. Acceptable but should be verified with `renderer.info`.

**Recommended change:**
1. Simplify road geometry: reduce vertex density for non-visible roads
2. Use `BufferGeometryUtils.mergeBufferGeometries` to reduce draw calls
3. Profile with `renderer.info` to verify memory usage

**Estimated impact:** ~2 hours for geometry simplification in the build script.

---

### 2.4 Animation Loop

#### Medium — No frame budget allocation

The `useFrame` loop handles vehicle position interpolation (~1000 objects), camera orbit animation, bloom post-processing, and raycasting for tooltips. Without a frame budget, these compete for the same 16.6ms slot.

Tooltip raycasting on 10K+ objects is O(n) and can spike to 5–10ms.

**Recommended change:**
1. Budget: interpolation (2ms), camera (1ms), raycasting (3ms max), render (10ms)
2. Use `drei/Bounds` or spatial indexing for raycasting
3. Debounce tooltip raycasting (only cast every 3rd frame)

**Estimated impact:** ~3 hours to implement raycasting optimization.

---

### 2.5 Memory Usage

#### Low — Vehicles outside the bounding box

ADSBx returns aircraft in a 25km radius. Some will be at 10,000m altitude over the edge of the map. Ships from AIS may be in the North Sea, not the Thames.

**Recommended change:** Filter vehicles to a tighter bounding box before rendering. For ships, filter to Thames coordinates only. For planes, filter to aircraft below 5,000m or within 15km.

**Estimated impact:** ~1 hour.

---

## 3. Backend Review

### 3.1 Caching

#### Medium — In-memory cache has no eviction policy

The architecture mentions "in-memory cache with TTL" but does not specify eviction. With continuous polling, the cache will grow unbounded if TTL does not trigger cleanup.

**Recommended change:** Use `lru-cache` or a simple Map with TTL-based cleanup. Set max entries per source.

**Estimated impact:** ~1 hour.

#### Medium — Backend state is ephemeral

The in-memory cache dies on restart. If the server restarts (deployment, crash, OOM), all polling state resets and the frontend sees an empty map until the cache warms up.

**Recommended change:** Accept it for V1 but add a "warming" indicator so the frontend knows the cache is cold. Plan for Redis or at-disk cache in V2.

**Estimated impact:** ~2 hours.

---

### 3.2 Retries

#### Medium — No retry strategy defined

The PRD mentions "retry logic" in Phase 14 but no algorithm. Naive retry (immediate re-request) will amplify rate limit problems.

**Recommended change:** Implement exponential backoff with jitter:

```
delay = min(baseDelay * 2^attempt + random(0, 1000), maxDelay)
```

Base: 1s, max: 30s, max attempts: 5. After 5 failures, enter circuit-breaker state.

**Estimated impact:** ~2 hours.

---

### 3.3 Rate Limiting

#### High — No client-side rate limiting on the proxy

The backend aggregates API calls but does not limit client requests. A single client polling every 1 second will waste server resources.

**Recommended change:**
1. Enforce minimum poll interval per client (e.g., 5s)
2. Use SSE so the server pushes updates instead of clients polling
3. Rate limit at the Express level with `express-rate-limit`

**Estimated impact:** ~2 hours.

---

## 4. Code Architecture Review

### 4.1 Module Boundaries

#### Medium — Backend and frontend in the same repo with no clear separation

The directory structure in `plan-architecture.md` shows everything under `src/`. There is no `server/` or `backend/` directory. The Express proxy is mentioned but not placed.

**Recommended change:**

```
london-live-map/
├── src/                    # Frontend (Vite + React)
│   ├── components/
│   ├── store/
│   ├── hooks/
│   ├── utils/
│   └── types/
├── server/                 # Backend (Express)
│   ├── adapters/           # API adapters
│   ├── routes/             # Express routes
│   ├── cache/              # Caching layer
│   └── inference/          # Position inference
├── build/                  # Build scripts (Parquet → glTF)
│   └── buildings.ts
├── data/                   # Static data output
│   └── buildings.glb
└── docs/
```

**Estimated impact:** ~1 hour restructuring.

---

### 4.2 Testing Strategy

#### Medium — No test framework installed

`package.json` has no test runner. `ADR-008` says "TDD for logic layer only" but there is no Vitest/Jest installed.

**Recommended change:** Add Vitest (matches Vite, zero config). Write tests for:

1. `geo.ts`: coordinate conversion (known London landmarks)
2. `routeInterpolation.ts`: position along route (edge cases: first/last point, empty route)
3. `inference.ts`: ETA → position mapping
4. `cache.ts`: TTL expiration, stale data

**Estimated impact:** ~3 hours setup + initial tests.

---

### 4.3 Interfaces

#### Low — No shared types between frontend and backend

The backend returns JSON. The frontend defines its own types. Without shared types, the two sides can drift.

**Recommended change:** Put shared types in a `packages/shared/` directory or a `types/` folder that both frontend and backend import. Since this is a monorepo, a `tsconfig` path alias works.

**Estimated impact:** ~1 hour.

---

## 5. Security

### 5.1 Credential Management

#### Medium — No API key rotation or secret management

`.env.local` is fine for local dev. There is no plan for production secret management. If this deploys to a PaaS (Render, Railway, Fly.io), keys need to be injected as env vars.

**Recommended change:**
- Verify `.env.local` is in `.gitignore` (it is, via `*.local` pattern)
- Add a pre-commit hook or `git secrets` rule to catch accidental commits
- Document the production env var names in `.env.example`
- Plan for platform-specific secret injection (Fly.io secrets, Render env vars)

**Estimated impact:** 30 minutes.

#### Low — No CORS or origin validation on the proxy

The Express proxy has no origin check. Any site can call `localhost:3001/api/tube` and burn through the API quota.

**Recommended change:** Add `Access-Control-Allow-Origin` header restricted to the frontend origin. In production, restrict to the deployed domain.

**Estimated impact:** 15 minutes.

---

## 6. Observability

### 6.1 Logging and Health Checks

#### Medium — No logging or health check endpoint

The backend has no `/health` endpoint, no request logging, and no way to know if polling is running.

**Recommended change:**
- `GET /health` — returns status of each data source (last fetch time, success/fail)
- Structured logging with `pino` or `winston`
- Log polling intervals and failure counts
- Log credit consumption for Darwin DBRS

**Estimated impact:** ~2 hours.

---

## 7. Legal / Compliance

### 7.1 Attribution

#### Medium — No attribution plan

OSM requires attribution ("© OpenStreetMap contributors"). TfL requires "Transport for London". Overture requires attribution. The PRD has no attribution UI component.

**Recommended change:** Add a small attribution footer or "About" panel listing all data sources with links. Include it in Phase 12 (UI Polish).

**Estimated impact:** 1 hour.

---

## 8. Implementation Plan Review

### 8.1 Build Order Assessment

The PRD's Phase order (Setup → Infrastructure → Loading → Map → Routes → Vehicles → API → Production) is mostly correct but has two problems.

#### High — Phase 14 (Real API Integration) is too late

Real API integration is the last substantive phase before production. This means:
- All vehicle rendering is tested with dummy data
- The inference engine is never validated against real data until the end
- API-specific bugs (missing fields, unexpected formats, duplicate entries) are discovered last

If TfL returns a different schema than documented, the entire vehicle rendering pipeline needs rework. Discovering this in Phase 14 wastes 3 weeks of work.

**Recommended change:** Integrate at least ONE real API in Phase 6 (Tube). Use real TfL data to validate the inference engine early. Keep dummy data for other layers until Phase 14, but prove the pipeline with real data first.

**Estimated impact:** Shift Phase 6 to use real TfL data. ~2 days earlier API work.

#### High — Build script for buildings (Phase 4) is a critical path blocker

Phase 4 depends on:
1. Overture tile availability (unknown)
2. Parquet → glTF conversion pipeline (not built)
3. Build script that may OOM on large tiles

If the build script fails or Overture data is incomplete, Phases 4–15 are blocked.

**Recommended change:**
1. Spike Overture tile availability NOW (before Phase 2)
2. Build a prototype conversion script for a 1km² test area
3. Validate the output renders correctly in Three.js
4. Only then proceed to full Greater London

**Estimated impact:** 2–3 days of spike work before Phase 4.

---

### 8.2 Dependencies Between Tasks

```
Phase 2 (Infrastructure) ──┬──→ Phase 3 (Loading Screen)
                           ├──→ Phase 4 (Map Foundation) ──→ Phase 5 (Route Geometry)
                           │                                    │
                           │                                    ├──→ Phase 6 (Tube) ──→ Phase 7 (Bus)
                           │                                    │                      │
                           │                                    │                      └──→ Phase 8 (Train)
                           │                                    │
                           │                                    └──→ Phase 9 (Planes)
                           │                                       │
                           │                                       ├──→ Phase 10 (Ships)
                           │                                       │
                           │                                       └──→ Phase 11 (River Boats)
                           │                                          │
                           │                                          └──→ Phase 12 (UI Polish)
                           │                                             │
                           │                                             └──→ Phase 13 (Jamcam)
                           │                                                │
                           │                                                └──→ Phase 14 (Real API)
                           │                                                   │
                           │                                                   └──→ Phase 15 (Production)
```

**Parallelizable work:**
- Phases 9, 10, 11 are independent after Phase 4+5 complete
- Phase 3 (Loading Screen) can run in parallel with Phase 4
- Backend proxy (Phase 2) can be built alongside frontend infrastructure

---

### 8.3 Risky Milestones

| Milestone | Risk | Mitigation |
|-----------|------|------------|
| Overture tile download + conversion | **Critical** — data may not exist for full London | Spike first with 1km² area |
| TfL API key acquisition | **Low** — free, instant | Register before Phase 6 |
| Overpass geometry fetch | **Medium** — rate-limited, query may timeout | Use overpass-turbo or pre-cached queries |
| Inference engine accuracy | **High** — linear interpolation may look wrong at junctions | Test with real TfL data early |
| Building merge script | **Medium** — may OOM on large datasets | Process zone-by-zone, stream output |
| Darwin credit exhaustion | **High** — unknown cost per call | Research credits before Phase 8 |

---

## 9. Edge Cases and Second-Order Concerns

### 9.1 Timezone Handling

#### Low — No timezone handling

The backend runs in one timezone. Darwin returns ISO timestamps. TfL returns local times. ADSBx returns Unix timestamps. Mixing them without explicit timezone handling causes off-by-one-hour errors in inference.

**Recommended change:** Normalize all timestamps to UTC internally. Convert to local time only in the UI tooltip.

**Estimated impact:** ~1 hour.

### 9.2 Data Quality Observability

There is no way to verify if "inferred" positions are accurate. A train inferred to be between two stations may actually be delayed at a signal.

**Recommended change (V2):** Implement a backend "heartbeat" monitor to flag stale entities (vehicles that have not moved despite ETA changes).

### 9.3 Empty Map User Experience

A 3D city is disorienting without guidance.

**Recommended change:** Implement preset camera views ("City Overview", "Westminster Focus", "Heathrow Corridor") and a "Center on Me" feature. Include in Phase 12.

### 9.4 Build Pipeline in CI/CD

`ADR-004`'s plan to convert Parquet to GeoJSON at build time will likely fail in CI/CD (GitHub Actions/Vercel) due to memory limits and deployment size caps.

**Recommended change:** Move map-data processing to a standalone data pipeline that uploads assets to a CDN/S3 bucket. The Vite build should reference pre-built assets, not generate them.

---

## 10. Missing ADRs

The following Architecture Decision Records should be created before implementation:

1. **ADR-009: Data Tiling and Streaming Strategy** — How to handle large-scale map data without crashing the browser (replaces ADR-004)
2. **ADR-010: Backend State Synchronization** — Caching, broadcasting, and SSE logic to solve rate limits (replaces ADR-006)
3. **ADR-011: Coordinate Transformation Pipeline** — Math for `EPSG:4326 → EPSG:3857 → Three.js World Space`
4. **ADR-012: Graceful Degradation** — UI behavior when specific APIs are down
5. **ADR-013: Inference Engine Location** — Backend vs frontend decision
6. **ADR-014: Deployment Platform** — Where the app runs and how secrets are managed

---

## 11. Go/No-Go Decision

**Conditional GO.** The project is buildable.

### Before Phase 2 starts:
- [ ] Replace ADR-004 with Draco-compressed glTF build pipeline
- [ ] Redesign backend from proxy to aggregator with SSE
- [ ] Pick deployment platform
- [ ] Spike Overture tile availability with 1km² test area

### In Sprint 1:
- [ ] Research Darwin credit costs
- [ ] Add Vitest + initial tests for geo.ts
- [ ] Define unified transport schema
- [ ] Restructure repo with server/ directory

### During implementation:
- [ ] Integrate real TfL API in Phase 6 (not Phase 14)
- [ ] Implement distance culling for buildings
- [ ] Add circuit breaker + per-layer status UI
- [ ] Add attribution footer

**Estimated timeline:** ~8–10 weeks for V1 with one developer, assuming the spike work and backend redesign are completed first.

---

*End of review.*
