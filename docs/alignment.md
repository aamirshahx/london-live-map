# London Live 3D Map — Alignment Record

This document records every design decision made during the planning phase.
Each entry includes the question, options considered, our choice, and rationale.

---

## Q1: Data Source Selection — National Rail

**Question:** Use TfL's own Live Running Information for National Rail, or Darwin DBRS?

**Options:**
- A: TfL Live Running Information (single API key, simpler)
- B: Darwin DBRS (richer data, separate account)

**Decision:** B — Use both TfL and Darwin. Keep Darwin for richer National Rail data.

**Rationale:** User wants both sources for maximum coverage. Two API keys is acceptable.

---

## Q2: Planes — ADSBx vs OpenSky

**Question:** Which API for aircraft tracking?

**Options:**
- A: OpenSky Network (free, 1 req/10s, less detailed)
- B: ADSBx v2 (free, 1 req/10s, more detailed, callsigns + airline)

**Decision:** B — ADSBx v2 API (`gateway.adsbexchange.com/api/aircraft/v2`)

**Rationale:** Better data quality (callsigns, airline names, aggregated from many receivers). Same rate limit. Requires API key registration.

---

## Q3: Ships — AIS Data Source

**Question:** Which API for vessel tracking?

**Options:**
- A: AIS Hub (free, 1 req/60s, username auth)
- B: MyShipTracking (free, 1 req/60s)
- C: MarineTraffic (paid)

**Decision:** A — AIS Hub with registered username

**Rationale:** Best data/price ratio. Thames has ~10-30 vessels; 60s refresh is visually fine. Requires registration at aishub.net.

---

## Q4: Map Data — Overture + Overpass

**Question:** What map data sources?

**Decision:** Overture Maps for 3D buildings + Overpass API for route geometry

**Rationale:** Overture provides 3D building footprints with height. Overpass provides road/rail/waterway geometry for transport animation. Both free, both open.

---

## Q5: Position Inference — Bus/Train Animation

**Question:** How detailed should position inference be?

**Options:**
- A: Simple linear interpolation
- B: Graph-based routing
- C: Hybrid — OSM geometry + nearest-point snapping

**Decision:** C — Hybrid approach

**Rationale:** OSM geometry gives actual road/rail paths including turns. Snapping to nearest point keeps vehicles on roads/tracks. Best balance of accuracy and complexity.

---

## Q6: Refresh Rates

**Question:** What polling intervals for each layer?

**Decision:**
- Planes: 10s (max allowed by ADSBx rate limit)
- Everything else: 30s
- Ships: 60s (max allowed by AIS Hub rate limit)
- Jamcams: 60s

**Rationale:** Planes move fastest, need most frequent updates. Ships move slowest. Animation runs at 60fps via requestAnimationFrame regardless of poll rate.

---

## Q7: Visual Style

**Question:** What visual style for the 3D scene?

**Decision:** Dark moody style, real TfL colors for tube lines, simple capsule/box vehicles, flat dark water, colored emissive neon lines with bloom post-processing

**Rationale:** Matches the reference screenshots. Dark theme with neon glow is visually striking and performant.

---

## Q8: UI — Tooltip Content

**Question:** What info to show on hover/click?

**Decision:** Minimal tooltips with essential info per transport type:
- Bus: route, registration, destination, next stops, ETA
- Tube: line, direction, next station, delay
- Train: operator, destination, next station, status
- Plane: callsign, airline, altitude, speed, heading
- Ship: name, MMSI, speed, heading, dimensions, flag

**Rationale:** Covers all essential info without clutter. Can be tweaked later.

---

## Q9: London Bounding Box

**Question:** How much area to cover?

**Decision:** Greater London (all zones), ~30km x 25km

**Rationale:** Full experience with all Tube lines, bus routes, Thames estuary, airport corridors, National Rail stations.

---

## Q10: Backend Architecture

**Question:** Pure frontend or minimal backend proxy?

**Decision:** B — Minimal Node.js/Express proxy server

**Rationale:** Solves CORS, keeps API keys server-side, simple to deploy, clean separation of concerns.

---

## Q11: Data Caching Strategy

**Question:** How to cache data?

**Decision:** In-memory cache with TTL per data type, stale-while-revalidate

**Rationale:** Simple, effective. TTL configurable per data type. Frontend never worries about rate limits.

---

## Q12: Credential Management

**Question:** How to handle API keys?

**Decision:** `.env.local` for actual keys (gitignored), `.env.example` for template (committed)

**Rationale:** Standard practice. Proxy server reads from process.env. Frontend never sees keys.

---

## Q13: Three.js Rendering Approach

**Question:** Instanced meshes vs individual meshes vs custom shaders?

**Decision:** Instanced meshes with per-instance colors via setColorAt()

**Rationale:** One draw call per transport type. ~10-15 total draw calls for all vehicles. Supports per-instance colors natively.

---

## Q14: Coordinate System

**Question:** How to map lat/lng to 3D world?

**Decision:** Web Mercator projection, London center at 51.5074, -0.1278, 1 unit = 1 meter

**Rationale:** Standard projection. Natural scale for Three.js. Simple offset to center.

---

## Q15: Overpass API Geometry

**Question:** What geometry to fetch from OSM?

**Decision:** Rail lines, bus routes, waterways, major roads, street lamps

**Rationale:** Covers all transport animation paths. ~50-100MB of data, fetched once at startup.

---

## Q16: State Management

**Question:** Zustand store structure?

**Decision:** Single store with Vehicle[] per type, selectedVehicle, hoveredVehicle, loading/error state

**Rationale:** Simple, flat structure. Components subscribe to slices they need.

---

## Q17: Build Tool

**Question:** Vite middleware vs separate processes?

**Decision:** Separate Vite + Express processes via `concurrently`

**Rationale:** Cleaner separation, easier to deploy later, Vite proxy handles CORS in dev.

---

## Q18: Overture Maps Processing

**Question:** Browser-side parsing vs backend parsing vs pre-processing?

**Decision:** Pre-processing — download Overture tiles once, convert Parquet → GeoJSON, serve static

**Rationale:** Map data is static infrastructure. Zero runtime cost. Keeps browser bundle small. What every 3D mapping app does.

---

## Q19: Development Order

**Question:** Which slice to build first?

**Decision:** Static UI with dummy data first, real APIs last

**Rationale:** Build visual foundation without being blocked by API keys. Validate design early. API integration is just a data layer swap at the end.

---

## Q20: React vs Alternatives

**Question:** Is React the right choice for a UI-heavy 3D app?

**Decision:** Yes — React + R3F

**Rationale:** R3F requires React. UI is lightweight (95% canvas). React overhead is negligible. Alternatives offer no meaningful advantage.

---

## Q21: Loading Screen

**Question:** Persistent or one-time?

**Decision:** One-time only on app launch

**Rationale:** Subsequent data updates are silent. Loading screen is for initial setup only.

---

## Q22: Neon Glow Effect

**Question:** How to achieve neon glow on transport lines?

**Decision:** UnrealBloomPass post-processing

**Rationale:** Simple, looks great, minimal performance cost (one extra pass). Matches screenshots perfectly.

---

## Q23: Tube Visualization

**Question:** Station dots or trains on tracks?

**Decision:** Trains on tracks — neon-colored lines with animated dots moving based on ETA

**Rationale:** Matches reference screenshots. More visually compelling. Requires tube line geometry from OSM.

---

## Q24: Loading Screen Design

**Question:** What should the loading screen look like?

**Decision:** Dark background, green neon (#00ff88), skyline SVG logo, "ZONE ONE" title, progress bars per layer

**Rationale:** Matches reference screenshots exactly. Professional, on-brand aesthetic.

---

## Q25: Vehicle Shapes

**Question:** Glowing dots vs 3D models?

**Decision:** Per transport type:
- Bus: red capsule/box
- Ship: teal capsule/box
- Train: glowing dot
- Plane: white capsule
- River boat: glowing dot
- Street lamp: small glowing yellow box

**Rationale:** Matches reference screenshots exactly. Simple shapes are performant and clean.

---

## Q26: Elevation Handling

**Question:** How to keep vehicles on roads and not fly over buildings?

**Decision:** Ground plane at Y=0, roads at Y=0.1, river at Y=-0.5, vehicles snap to road/track geometry

**Rationale:** Vehicles constrained to geometry path. Can't leave the path because we interpolate along the path array.

---

## Q27: Jamcam Feature

**Question:** Real or mock TFL Jamcam overlay?

**Decision:** Real — uses TfL API endpoint `/Place/Type/JamCam`

**Rationale:** Found working API endpoint. Returns camera positions, image URLs, video URLs. No mock needed.

---

## Q28: Jamcam Trigger

**Question:** What triggers the Jamcam overlay?

**Decision:** Hover over traffic signals (street lamps), not road segments

**Rationale:** Reference screenshots show jamcam appearing when hovering over signal posts. Signals are decorative elements placed along roads.

---

## Q29: Street Lamps / Signals

**Question:** Real OSM street lamps or procedural placement?

**Decision:** Procedurally placed along major roads

**Rationale:** OSM street lamp data is sparse. Procedural placement looks more complete and matches screenshots.

---

## Q30: Plane Orientation

**Question:** How to orient planes in 3D?

**Decision:** Rotate by `true_track`, slight pitch from `vert_rate`

**Rationale:** Uses ADSBx data directly. Simple, accurate enough at this scale.

---

## Q31: Ship Orientation

**Question:** How to orient ships in 3D?

**Decision:** Rotate by `heading`, no pitch/roll

**Rationale:** Ships don't tilt much visually at this scale. Heading is the most relevant orientation data.

---

## Q32: Bus Orientation

**Question:** How to orient buses in 3D?

**Decision:** Face forward along road geometry (next point on path)

**Rationale:** Buses follow road paths. Orientation determined by geometry direction, not API data.

---

## Q33: Train Orientation

**Question:** How to orient trains in 3D?

**Decision:** Face forward along rail geometry (next point on path)

**Rationale:** Trains follow rail paths. Orientation determined by geometry direction.

---

## Q34: Build Order

**Question:** Final build order?

**Decision:**
1. Scaffold + project setup
2. Loading screen
3. Map foundation (buildings + water + ground)
4. Street lamps + signals
5. Tube lines (neon + animated dots)
6. Bus markers (capsules + inference)
7. Backend proxy skeleton (dummy data)
8. Train markers (dots on rail)
9. Plane markers (capsules)
10. Ship markers (capsules)
11. River boats (dots on waterway)
12. UI polish (tooltips, toggles, reset)
13. Real API integration
14. Jamcam overlay
15. Production build

**Rationale:** Builds visual foundation first, adds complexity incrementally, API swap at the end.

---

## Final Score

| Criteria | Score |
|----------|-------|
| Feasibility | 9/10 |
| Data freshness | 9/10 |
| Visual fidelity | 9/10 |
| Complexity | 7/10 |
| Performance | 8/10 |
| Scope clarity | 10/10 |

**Overall: 8.8/10**

## Remaining Risks (Known, Acceptable)

1. **OSM geometry gaps** — Some bus routes/rail spurs may be missing from Overpass. Handle gracefully with fallback to straight-line animation.
2. **Rate limit edge cases** — APIs may be slow or return stale data. Proxy cache handles this with stale-while-revalidate.
3. **Input validation** — Bad coordinates or missing fields from APIs. Add validation during API integration phase.
4. **Error handling** — API failures, corrupt data, network issues. Add retry logic and graceful degradation during API integration.
5. **Deployment** — No production build/deploy plan yet. Address in Phase 15.

---

## Post-Review Alignment (Pass 2)

Decisions made after incorporating 3 independent architecture reviews (Gemma 4 31B, Qwen 3.6 27B, ChatGPT).

### Q35: Building Data Delivery — GeoJSON vs Draco glTF

**Question:** How to deliver 400-800MB of Overture building data to the browser?

**Reviewer consensus:** Static GeoJSON will crash the browser. JSON.parse freezes main thread for 10-30s. Heap expansion 2-4x raw size. 50k-100k buildings exceed practical InstancedMesh limits.

**Decision:** Download Overture Parquet tiles, convert to single merged glTF with Draco compression at build time. Target <50MB compressed. Load via `useGLTF` from Drei. Three.js handles LOD and frustum culling internally.

**Rationale:** All 3 reviewers flagged this as Critical. The browser cannot handle 800MB of JSON. Draco glTF is the minimum viable fix that keeps the stack simple.

---

### Q36: Backend Architecture — Proxy vs Aggregator

**Question:** Should the backend be a proxy (forwarding client requests) or an aggregator (polling on its own schedule)?

**Reviewer consensus:** Proxy model exhausts API rate limits with 2+ concurrent users. Each client triggers separate API calls.

**Decision:** Backend is an Aggregator. It polls APIs on its own schedule, maintains canonical vehicle state via a Vehicle State Engine, and broadcasts to all clients via SSE (Server-Sent Events).

**Rationale:** One poll serves all users. Rate limits are respected regardless of client count. SSE is push-based, lighter than HTTP polling.

---

### Q37: Vehicle Position Storage — Zustand vs Mutable Refs

**Question:** Where should vehicle positions be stored?

**Reviewer consensus:** Updating 1000+ vehicles through Zustand every 10-30s triggers React re-renders of all components. Even with `useShallow`, the reconciliation cycle causes frame drops.

**Decision:** Vehicle positions stored in mutable refs. `useFrame` reads refs directly — no React re-render triggered. Only UI state (hover, click, search, layer toggles) stored in Zustand.

**Rationale:** Reviewer 1: "React render cycle overhead." Reviewer 2: "Zustand store will thrash." The renderer should never care about polling cadence. Poll → update simulation state → interpolate locally → render snapshots.

---

### Q38: Inference Engine Location

**Question:** Should the inference engine run on the backend or frontend?

**Reviewer 3:** "The inference engine section is massively underestimated." TfL arrivals are station-centric, not train-graph. Darwin doesn't provide enough info for continuous train placement.

**Decision:** Inference engine runs on the backend. Frontend receives render-ready snapshots (x,y,z coordinates). Backend holds map geometry.

**Rationale:** Reviewer 1: "Move the Inference Engine to the backend." Consistent data across all clients. Frontend is simpler (just renders).

---

### Q39: Unified Transport Schema

**Question:** How to handle 5 different external API response formats?

**Reviewer 1:** "Brittle API Data Contracts. A single field change in the TfL API could break the entire transport layer."

**Decision:** All adapters output the same `TransportEntity` shape. Frontend consumes only this schema. Adapter pattern makes provider swapping easy.

**Rationale:** Frontend isolated from API changes. System resilience.

---

### Q40: Entity Resolution

**Question:** How to handle vehicle identity across polling cycles?

**Reviewer 3:** "Not discussed in the docs. Without stable identifiers you get despawn/respawn every polling cycle."

**Decision:** Backend Entity Resolver matches incoming vehicles to existing entities using stable IDs (icao24, mmsi, registration) or route+direction+time windows.

**Rationale:** Smooth vehicle transitions. No flickering.

---

### Q41: Visual Distinction — GPS vs Inferred

**Question:** How to differentiate GPS-verified vs inferred positions?

**Reviewer 1:** "Users may mistake an inferred position for a precise GPS coordinate."

**Decision:** `dataCertainty: 'gps' | 'inferred'` field. GPS vehicles: solid glow. Inferred vehicles: slightly dimmer glow or dashed trail.

**Rationale:** Transparency about data quality. Trust.

---

### Q42: Filter/Search for Vehicles

**Question:** How to handle thousands of vehicles becoming a "noise cloud"?

**Reviewer 1:** "No requirement for filtering by line or searching for specific vehicle IDs."

**Decision:** Add SearchPanel component with search bar and filter results. Filter by transport type, line, route.

**Rationale:** Transforms from passive art piece to functional tool.

---

### Q43: Attribution Overlay

**Question:** How to comply with Overture, OSM, TfL attribution requirements?

**Reviewer 1:** "Strict attribution requirements."

**Decision:** Persistent footer showing data source credits.

**Rationale:** Legal compliance. Minimal UI impact.

---

### Q44: Deployment Strategy

**Question:** Where and how to deploy?

**Reviewer 2:** "Pick a deployment platform now, not in Phase 15."

**Decision:** Single server (Express serves static files + API routes). Deployed via Docker or direct to VPS. Simpler than separate hosting.

**Rationale:** Fewer moving parts. Easier debugging.

---

### Q45: Darwin Credit Accounting

**Question:** How to account for Darwin DBRS credit costs?

**Reviewer 2:** "Unknown cost per call."

**Decision:** Estimated ~200 calls/hour × 16 hours = 3,200 credits/day. Well within 100k daily limit. Monitored via Heartbeat Monitor.

**Rationale:** No risk of credit exhaustion with current polling schedule.

---

### Q46: Build Pipeline for Map Data

**Question:** How to handle Overture tile download + conversion in CI/CD?

**Reviewer 1 & 2:** "Will likely fail in CI/CD due to memory limits and deployment size caps."

**Decision:** Run build scripts locally (not in CI/CD). Output saved to `public/map-data/`. Committed to repo. Vite build references pre-built assets.

**Rationale:** CI/CD memory limits are too low for Parquet → glTF conversion. Local build is the practical approach.

---

### Q47: Real API Integration Timing

**Question:** Should real API integration be Phase 14 (last) or earlier?

**Reviewer 2:** "Integrate real TfL API in Phase 6 (not Phase 14)."

**Decision:** Keep dummy data for development. Integrate real APIs during Phase 14 but validate with real data during Phase 5-6 (inference engine testing).

**Rationale:** Dummy data allows visual iteration. But inference logic must be tested with real data early. Hybrid approach: dummy for UI, real for inference validation.

---

### Q48: OSM Bus Route Data Quality

**Question:** How to handle inconsistent OSM bus route data?

**Reviewer 3:** "OSM bus route data quality varies wildly. Some routes are perfect; others are broken, incomplete, or disconnected."

**Decision:** Spike-test Overpass queries early. Validate geometry. For broken routes, fall back to straight-line interpolation between stops.

**Rationale:** Reality check on OSM data quality. Graceful degradation for incomplete data.

---

### Q49: Tube Train Placement Accuracy

**Question:** How to place Tube trains accurately when TfL arrivals are station-centric?

**Reviewer 3:** "Reconstructing actual train positions is much harder than the PRD implies."

**Decision:** For lines with good OSM geometry, use ETA-based interpolation. For lines with poor geometry, fall back to station-dot visualization. Test with real TfL data early.

**Rationale:** "The map isn't the hard part. Reality is." Tube train placement is the highest-risk inference task.

---

### Q50: Heartbeat Monitor

**Question:** How to verify inferred positions are accurate?

**Reviewer 1 & 2:** "No way to verify if 'Inferred' positions are accurate."

**Decision:** Backend Heartbeat Monitor flags vehicles that haven't moved despite ETA changes. Logs warnings. Helps debug inference accuracy.

**Rationale:** Data quality observability.

---

## Updated Score

| Criteria | v1 Score | v2 Score | Change |
|----------|----------|----------|--------|
| Feasibility | 9/10 | 8.5/10 | -0.5 (inference complexity revealed) |
| Data freshness | 9/10 | 9/10 | No change |
| Visual fidelity | 9/10 | 9/10 | No change |
| Complexity | 7/10 | 6/10 | -1 (more complexity revealed) |
| Performance | 8/10 | 9/10 | +1 (mutable refs + Draco glTF fix critical issues) |
| Scope clarity | 10/10 | 9/10 | -1 (inference uncertainty acknowledged) |

**Overall v2: 8.5/10**

**Key insight from reviewers:** "The biggest risk is not the UI, React, Three.js, Zustand, or performance. The biggest risk is: 'If I know where stations are and know where tracks are, I can know where trains are.' That sounds reasonable until you try implementing it."

**Mitigation:** Spike-test inference engine with real TfL data during Phase 5. If inference fails, fall back to station-dot visualization. The map and UI remain functional regardless.
