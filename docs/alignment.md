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
