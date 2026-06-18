# Architecture Review: Pass 1
**Date:** 2026-06-18
**Reviewer:** Principal Product Architect / Senior Software Reviewer
**Model** Gemma 4 31B

## Executive Summary
The "Zone One" project has a strong conceptual foundation and a clear data strategy. However, it currently contains two **Critical** architectural flaws regarding data delivery and API orchestration that will lead to browser crashes and API bans if not addressed before implementation. The transition from a "visualization" to a "product" requires better UX journeys (filtering/search) and a more robust data pipeline.

---

## 1. Product Review

### Missing User Journeys & UX
- **Severity: Medium** | **Problem: Lack of Search/Filter Capabilities**
  - **Issue:** With thousands of vehicles, the map will become a "noise cloud." There is no requirement for filtering by line (e.g., "Central Line only") or searching for specific vehicle IDs/Callsigns.
  - **Recommendation:** Implement a UI filter panel and a search function.
  - **Impact:** Transforms the project from a passive art piece into a functional tool.

- **Severity: Low** | **Problem: Visual Ambiguity of Data Certainty**
  - **Issue:** The system mixes GPS-live data (Planes/Ships) with inferred data (Buses/Trains). Users may mistake an inferred position for a precise GPS coordinate.
  - **Recommendation:** Use distinct visual cues (e.g., solid vs. dashed trails or different glow intensities) to differentiate GPS-verified vs. inferred positions.
  - **Impact:** Increases transparency and trust in the data.

- **Severity: Medium** | **Problem: "Real-time" Expectation vs. Polling Latency**
  - **Issue:** The tagline promises "real time," but 30s polling intervals will cause vehicles to "jump" visually.
  - **Recommendation:** Explicitly require client-side interpolation to smoothly animate vehicles between polling updates.
  - **Impact:** Ensures the "Live" feeling promised in the product vision.

---

## 2. Architecture Review

### Structural Flaws
- **Severity: Critical** | **Problem: Massive Frontend Data Payload (The 800MB GeoJSON Problem)**
  - **Issue:** `ADR-004` suggests serving static GeoJSON for buildings/roads (est. 400-800MB). Modern browsers cannot parse or store this volume of JSON in memory without crashing or freezing.
  - **Recommendation:** Abandon static GeoJSON for large-scale geometry. Implement **3D Tiles (OGC)** or a **Vector Tile** strategy to load geometry based on the camera frustum and zoom level.
  - **Impact:** Prevents browser crashes and reduces initial load time from minutes to seconds.

- **Severity: Critical** | **Problem: API Rate Limit Exhaustion (Proxy vs. Aggregator)**
  - **Issue:** `ADR-006` uses an Express Proxy. If the proxy simply forwards requests, the system will hit TfL/National Rail rate limits as soon as multiple users connect.
  - **Recommendation:** Pivot the backend from a **Proxy** to an **Aggregator/Cache**. The backend should poll APIs on its own schedule, store state in a cache (e.g., Redis), and serve that cache to all clients.
  - **Impact:** Enables scaling to thousands of users while remaining within API limits.

- **Severity: Medium** | **Problem: Under-engineered Inference Engine**
  - **Issue:** The logic for placing vehicles on geometry based on ETA is not explicitly located. Doing this on the frontend wastes CPU across all clients; doing it on the backend requires the backend to possess the map geometry.
  - **Recommendation:** Move the Inference Engine to the backend. Calculate final `(x, y, z)` coordinates server-side and send only the coordinates to the frontend.
  - **Impact:** Reduces frontend CPU load and ensures data consistency across all users.

---

## 3. Risk Review

### Technical & Operational Risks
- **Severity: High** | **Problem: GPU Memory (VRAM) Overflow**
  - **Issue:** Rendering every building in London will exceed VRAM on most consumer hardware.
  - **Recommendation:** Implement a strict **LOD (Level of Detail)** system and **Frustum Culling**. Use simplified bounding boxes for distant buildings.
  - **Impact:** Ensures the app runs on standard laptops/phones.

- **Severity: High** | **Problem: Brittle API Data Contracts**
  - **Issue:** Reliance on 5+ external APIs. A single field change in the TfL API could break the entire transport layer.
  - **Recommendation:** Implement an **Adapter Pattern** in the backend. Create a "Unified Transport Schema" that the frontend consumes, isolating it from external API changes.
  - **Impact:** Increases system resilience and allows for easier provider swapping.

- **Severity: Medium** | **Problem: React Render Cycle Overhead**
  - **Issue:** Updating thousands of vehicles via Zustand every few seconds may trigger excessive React re-renders, causing frame drops.
  - **Recommendation:** Use **Transient Updates** for vehicle positions. Update Three.js objects directly in the `useFrame` loop rather than triggering React state updates for every coordinate change.
  - **Impact:** Maintains a consistent 60 FPS.

---

## 4. Missing ADRs

The following Architecture Decision Records must be created:
1. **ADR-009: Data Tiling & Streaming Strategy** (How to handle 800MB of map data).
2. **ADR-010: Backend State Synchronization** (Caching/Broadcasting logic to solve rate limits).
3. **ADR-011: Coordinate Transformation Pipeline** (Math for `EPSG:4326` $\rightarrow$ `EPSG:3857` $\rightarrow$ `Three.js World Space`).
4. **ADR-012: Graceful Degradation** (UI behavior when specific APIs are down).

---

## 5. Second-Order Concerns

### Build Pipeline Bottleneck
`ADR-004`'s plan to convert Parquet to GeoJSON at build time will likely fail in CI/CD (GitHub Actions/Vercel) due to memory limits and deployment size caps.
**Recommendation:** Move map-data processing to a standalone data pipeline that uploads assets to a CDN/S3 bucket.

### Data Quality Observability
There is no way to verify if "Inferred" positions are accurate.
**Recommendation:** Implement a backend "Heartbeat" monitor to flag stale entities (e.g., vehicles that haven't moved despite ETA changes).

### "Empty Map" User Experience
A 3D city is disorienting without guidance.
**Recommendation:** Implement **Preset Camera Views** (e.g., "City Overview," "Westminster Focus") and a "Center on Me" feature.

### Legal & Attribution Compliance
Overture, OSM, and TfL have strict attribution requirements.
**Recommendation:** Add a persistent **Attribution Overlay** in the UI listing all active data sources.
