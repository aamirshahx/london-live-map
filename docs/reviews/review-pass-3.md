# Review Pass 3 — ChatGPT Analysis

> Source: ChatGPT review of `alignment.md`, `prd.md`, and `plan-architecture.md`.

---

## Biggest Strength

The strongest document is **Alignment.md**.

Most teams skip this. Six weeks later nobody remembers why Redis was chosen, why ADSBx was picked, or why a proxy server exists. Alignment.md prevents that.

**Verdict:** Keep this pattern. For future projects generate in this order:

1. Architecture
2. Alignment
3. PRD
4. ADRs (Architecture Decision Records)
5. Task breakdown

---

## Biggest Weakness

The model repeatedly assumes data exists that probably doesn't. Classic open-model behavior.

### Tube Trains

PRD claims:
- Fetch arrivals for all Tube stations
- Place trains on line geometry

**Problem:** TfL arrivals are station-centric. You get arrival predictions at stations, not a complete train graph. Reconstructing actual train positions is much harder than the PRD implies. The inference engine section is massively underestimated.

### National Rail

PRD says:
- Fetch departure boards
- Calculate train position

**Problem:** Darwin does not provide enough information to reliably place every train continuously on tracks. You'd need:
- Schedule data
- Route data
- Station sequence
- Delay propagation

The model treats this as a solved problem. It isn't.

### Bus Routes

The Overpass assumption is dangerous:

```
way["route"="bus"]
```

OSM bus route data quality varies wildly. Some routes are perfect; others are broken, incomplete, or disconnected. Validation tooling is needed.

---

## Biggest Technical Mistake — Memory Budget

PRD claims:
- Buildings: 200MB
- Geometry: 50MB
- Total: 272MB

For a browser app this is enormous and likely wrong. Loading 100,000 buildings as GeoJSON into the browser is one of the least efficient approaches possible. A serious implementation would use:
- Vector tiles
- Binary formats
- PMTiles
- MVT
- Custom packed geometry

Loading hundreds of MB into React converts an M2 Mac into a decorative space heater.

---

## Second Big Mistake — Draw Calls

Estimated draw calls:
- Tube lines: 13
- Bus routes: 100
- Rail: 20
- Roads: 50

That's **183 draw calls** before scene complexity. Realistically, line rendering, bloom, labels, tooltips, and overlays push this higher. Not fatal, just optimistic.

---

## Third Big Mistake — Overture Strategy

PRD says:
- Download Greater London (400–800MB)
- Convert to GeoJSON
- Serve static

This creates huge startup downloads, huge parsing cost, and huge memory usage for little gain. Better approach:
- PMTiles + MapLibre + vector tile pipeline
- Preprocess buildings into binary chunks

---

## Architectural Issue

The project is two systems pretending to be one:

**System 1 — Map Renderer:** buildings, roads, water, camera

**System 2 — Transport Intelligence Platform:** route inference, caching, polling, normalization, entity tracking

The PRD treats them equally. System 2 is 80% of the complexity. The map is the easy part. The transport inference layer is where weeks will be spent.

---

## Missing Layer

Current design:
```
API → Renderer
```
(with inference sprinkled around)

Needed:
```
Raw APIs
    ↓
Normalization
    ↓
Vehicle State Engine
    ↓
Renderer
```

Recommended server structure:
```
server/
  adapters/
    tfl.ts
    darwin.ts
    adsbx.ts
    ais.ts
  normalizers/
    transport.ts
  tracking/
    vehicleTracker.ts
  inference/
    trainInference.ts
    busInference.ts
```

Without this, the frontend absorbs business logic and React becomes the accidental backend.

---

## Missing Risk — Identity Reconciliation

Not discussed in the docs. Example: a train appears at Station A, then Station B. How do you know it's the same train? Without stable identifiers you get despawn/respawn every polling cycle. An **Entity Resolution Layer** is needed before rendering. This is a major omission.

---

## Missing Performance Strategy

PRD assumes:
```
poll → update store → rerender
```

For buses, trains, planes, and ships, a better approach:
```
poll → update simulation state → interpolate locally → render snapshots
```

The renderer should never care about polling cadence.

---

## Overall Assessment

| Document | Score |
|---|---|
| Architecture Plan | 8.5/10 |
| Alignment Record | 9.5/10 |
| PRD | 7.5/10 |
| Entire Planning Session | 8.5/10 |

**Key takeaway:** The biggest risk is not the UI, React, Three.js, Zustand, or performance. The biggest risk is:

> "If I know where stations are and know where tracks are, I can know where trains are."

That sounds reasonable until you try implementing it.

**Recommendation:** Freeze the UI/UX portions as-is. Keep the alignment document. Spend the next planning iteration validating data availability and transport inference assumptions. If those survive scrutiny, the rest is straightforward engineering. If they don't, half the PRD needs rewriting before a meaningful train appears on the map.

The map isn't the hard part. Reality is.
