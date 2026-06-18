# Software Development Lifecycle — Zone One

This document defines the SDLC process for the Zone One project.
All LLM agents working on this project MUST follow this lifecycle.

## Lifecycle Stages

```
1. Product Requirements Document (PRD)
        |
        ↓
2. Alignment Record
        |
        ↓
3. Architecture Proposal
        |
        ↓
4. Independent Architecture Reviews
        |
        +----------------+
        |                |
     Gemma 31B       Qwen 27B Dense
        |                |
        +----------------+
                 |
                 ↓
5. Architecture Revision
        |
        ↓
6. ADRs
        |
        ↓
7. Risk Register
        |
        ↓
8. Assumption Validation Plan / Spikes
        |
        ↓
9. Implementation Plan
        |
        ↓
10. Epic → Story → Task Breakdown
        |
        ↓
11. Coding
```

## Stage Descriptions

### Stage 1: Product Requirements Document (PRD)

**Purpose:** Define what the product does, who it's for, and the requirements.

**Deliverables:**
- `docs/prd.md` — Full PRD with features, APIs, UI specs, data sources, performance targets

**Rules:**
- PRD must be reviewed by the user before proceeding
- Include all data sources with exact endpoints, rate limits, auth
- Include UI/UX specifications
- Include performance targets and constraints
- Include development phases

### Stage 2: Alignment Record

**Purpose:** Record every design decision made during planning. Questions, options, choices, rationale.

**Deliverables:**
- `docs/alignment.md` — All Q&A decisions with rationale

**Rules:**
- Ask questions one at a time (grill-me approach)
- Provide recommended answer for each question
- Record the user's decision and rationale
- Include all 30+ decisions that shape the architecture

### Stage 3: Architecture Proposal

**Purpose:** Define the technical blueprint. Not a repeat of the PRD — concrete engineering decisions.

**Deliverables:**
- `docs/architecture.md` (v1) — System context, architecture, components, data flow, rendering, backend, deployment, security, ADRs

**Rules:**
- Must be detailed enough that a senior engineer can implement without asking questions
- Include concrete decisions, not vague descriptions
- Identify assumptions
- Highlight unresolved questions
- Include ADRs for major decisions

### Stage 4: Independent Architecture Reviews

**Purpose:** Stress-test the architecture with independent reviewers before implementation.

**Process:**
1. Send PRD + Alignment + Architecture to Gemma 4 31B (or equivalent)
2. Send PRD + Alignment + Architecture to Qwen 3.6 27B (or equivalent)
3. Optionally send to ChatGPT for additional review
4. Collect all review feedback

**Deliverables:**
- `docs/reviews/review-pass-1.md` — Gemma review
- `docs/reviews/review-pass-2.md` — Qwen review
- `docs/reviews/review-pass-3.md` — ChatGPT review

**Rules:**
- Reviews must be independent (no cross-contamination)
- Each review must cover: critical issues, high risks, medium concerns, missing ADRs
- Reviews must provide severity ratings
- Reviews must include go/no-go decision

### Stage 5: Architecture Revision

**Purpose:** Incorporate review feedback into the architecture. Fix critical and high severity issues.

**Process:**
1. Read all review feedback
2. Synthesize common issues across reviewers
3. Categorize by severity (Critical, High, Medium, Low)
4. Fix all Critical issues before implementation
5. Fix all High issues before implementation
6. Document Medium/Low issues for implementation phase

**Deliverables:**
- `docs/architecture.md` (v2) — Revised architecture
- `docs/alignment.md` — Updated with post-review decisions

**Rules:**
- Must address every Critical issue from every reviewer
- Must address every High issue from at least 2 reviewers
- Document what was changed and why
- Update ADRs as needed

### Stage 6: ADRs

**Purpose:** Formalize all architecture decisions with context, decision, and consequences.

**Deliverables:**
- Embedded in `docs/architecture.md` (ADR section)

**Rules:**
- Every ADR must have: Status, Context, Decision, Consequences
- ADRs must be numbered sequentially
- Include all decisions from v1 and v2 reviews
- New ADRs from review feedback must be added

### Stage 7: Risk Register

**Purpose:** Document all known risks, their severity, and mitigation strategies.

**Deliverables:**
- `docs/risk-register.md` — All risks with severity, probability, impact, mitigation, owner

**Risk Categories:**
- Technical risks (API failures, data quality, performance)
- Operational risks (rate limits, credits, deployment)
- Schedule risks (build blockers, spike work)
- Scope risks (inference complexity, data availability)

**Rules:**
- Each risk must have: ID, description, severity, probability, impact, mitigation, status
- Risks must be reviewed at each lifecycle stage
- New risks discovered during implementation must be added

### Stage 8: Assumption Validation Plan / Spikes

**Purpose:** Validate risky assumptions before implementation begins.

**Process:**
1. List all assumptions from the architecture document
2. Prioritize by risk (highest risk first)
3. Design spike tasks to validate each assumption
4. Execute spikes and document results

**Spike Examples:**
- Spike 1: Download 1km² of Overture tiles, convert to glTF, measure size and load time
- Spike 2: Query Overpass for London bus routes, validate data quality
- Spike 3: Fetch real TfL arrivals data, test inference engine with actual data
- Spike 4: Test ADSBx API coverage for London airspace
- Spike 5: Test AIS Hub coverage for Thames area

**Deliverables:**
- `docs/spikes.md` — Spike plan, execution results, conclusions

**Rules:**
- Spikes must be executed BEFORE implementation starts
- Results must be documented
- If a spike fails, architecture must be revised

### Stage 9: Implementation Plan

**Purpose:** Break the architecture into implementable phases with clear ordering and dependencies.

**Deliverables:**
- `docs/implementation-plan.md` — Phases, tasks, dependencies, estimates

**Rules:**
- Must follow the build order established in Alignment
- Dummy-first approach: static UI with dummy data before real APIs
- Each phase must have clear entry/exit criteria
- Dependencies between phases must be explicit
- Parallelizable work must be identified

### Stage 10: Epic → Story → Task Breakdown

**Purpose:** Break each implementation phase into user stories and actionable tasks.

**Process:**
1. For each phase, define Epics (major features)
2. For each Epic, define Stories (user-facing functionality)
3. For each Story, define Tasks (implementation steps)
4. Each Task must be small enough to complete in 1-2 hours

**Deliverables:**
- `docs/stories.md` — Epics, Stories, Tasks with status tracking

**Rules:**
- Tasks must be testable (pass/fail criteria)
- Tasks must be ordered by dependency
- TDD for pure logic modules (geo, inference, cache)
- Implement-and-test for components and API fetchers

### Stage 11: Coding

**Purpose:** Implement the stories and tasks.

**Process:**
1. Start with Phase 1 (Setup)
2. Complete each phase before moving to the next
3. Mark tasks as complete in `docs/stories.md`
4. Update Risk Register if new risks emerge
5. Update Architecture if design decisions change

**Rules:**
- Follow TDD for logic modules
- Follow implement-and-test for UI/components
- Commit frequently with descriptive messages
- Update documentation as code changes
- Never skip the dummy-data-first approach
- Real API integration only in the final phase

## Agent Instructions

When working on this project, LLM agents MUST:

1. **Check the current lifecycle stage** — Look at what documents exist and what stage we're at
2. **Never skip stages** — Don't start coding (Stage 11) without completing Stages 1-10
3. **Read all prior documents** — Before writing anything, read PRD, Alignment, Architecture, and reviews
4. **Follow the build order** — Respect the phase ordering in the Implementation Plan
5. **Update documentation** — Every decision, every change, every risk must be documented
6. **Ask before proceeding** — If a stage is unclear, ask the user for clarification
7. **Use the grill-me approach** — When making design decisions, ask one question at a time with recommendations
8. **Respect TDD scope** — TDD for logic only, implement-and-test for UI/components
9. **Dummy-first** — Never integrate real APIs until the final phase
10. **Document everything** — If it's not in the docs, it didn't happen

## Document Index

| Document | Stage | Path |
|----------|-------|------|
| PRD | 1 | `docs/prd.md` |
| Alignment | 2 | `docs/alignment.md` |
| Architecture | 3, 5 | `docs/architecture.md` |
| Reviews | 4 | `docs/reviews/review-pass-*.md` |
| ADRs | 6 | Embedded in `docs/architecture.md` |
| Risk Register | 7 | `docs/risk-register.md` |
| Spikes | 8 | `docs/spikes.md` |
| Implementation Plan | 9 | `docs/implementation-plan.md` |
| Stories | 10 | `docs/stories.md` |
| SDLC | All | `docs/sdlc.md` |

## Current Status

| Stage | Status | Documents |
|-------|--------|-----------|
| 1. PRD | ✅ Complete | `docs/prd.md` |
| 2. Alignment | ✅ Complete | `docs/alignment.md` |
| 3. Architecture | ✅ Complete (v2) | `docs/architecture.md` |
| 4. Reviews | ✅ Complete (3 reviews) | `docs/reviews/review-pass-{1,2,3}.md` |
| 5. Architecture Revision | ✅ Complete | `docs/architecture.md` (v2) |
| 6. ADRs | ✅ Complete | 14 ADRs in `docs/architecture.md` |
| 7. Risk Register | ⏳ Pending | — |
| 8. Spikes | ⏳ Pending | — |
| 9. Implementation Plan | ⏳ Pending | — |
| 10. Stories | ⏳ Pending | — |
| 11. Coding | ⏳ Pending | — |

**Next stage: 7. Risk Register**
