---
stepsCompleted: ['step-01-document-discovery', 'step-02-prd-analysis', 'step-03-epic-coverage', 'step-04-ux-alignment', 'step-05-epic-quality', 'step-06-final-assessment']
documents:
  prd: '_bmad-output/planning-artifacts/prd.md'
  architecture: 'docs/architecture/design-plan.md (pre-ceremony draft)'
  epics: 'docs/epics/epics.md (pre-ceremony draft)'
  ux: 'N/A (headless daemon)'
  brief: 'docs/bmad/product-brief.md'
  research: 'docs/research/did-provenance-research.md'
---

# Implementation Readiness Assessment Report

**Date:** 2026-03-02
**Project:** RustyClaw

## Document Inventory

| Type | File | Status |
|------|------|--------|
| PRD | `_bmad-output/planning-artifacts/prd.md` (41K) | ✅ Complete (steps 1-12) |
| Architecture | `docs/architecture/design-plan.md` (31K) | ⚠️ Pre-ceremony draft |
| Epics | `docs/epics/epics.md` (19K) | ⚠️ Pre-ceremony draft |
| UX Design | N/A | N/A (headless daemon) |

## PRD Analysis

### Functional Requirements (37 total)

**Message Routing (6 FRs)**
- FR1: Jay can send a message via Signal and receive a response from Canti
- FR2: Jay can see a typing indicator while Canti is composing a response
- FR3: RustyClaw can queue inbound messages when Canti is busy processing another request
- FR4: RustyClaw can prioritize interactive messages over scheduled/cron messages
- FR5: Jay can be authenticated via sender allowlist before messages are processed
- FR6: RustyClaw can reject messages from unauthorized senders silently

**Session Lifecycle (9 FRs)**
- FR7: RustyClaw can create a new Claude session with assembled system prompt
- FR8: RustyClaw can resume an existing Claude session by persisted UUID
- FR9: RustyClaw can persist session UUID to disk so sessions survive restarts
- FR10: RustyClaw can monitor context usage from Claude response metrics
- FR11: RustyClaw can inject system messages at soft threshold to encourage memory writes
- FR12: RustyClaw can enforce pre-rotation memory flush (instruct → wait for confirmation → rotate)
- FR13: RustyClaw can rotate sessions with summary injection into the new session
- FR14: RustyClaw can perform daily scheduled rotation at configured time
- FR15: RustyClaw can flush memory on graceful shutdown (SIGTERM)

**Memory Persistence — Declarative (5 FRs)**
- FR16: RustyClaw can assemble system prompts from configured markdown files (SOUL.md, USER.md, MEMORY.md, daily files)
- FR17: RustyClaw can index memory files into pgvector embeddings as files change
- FR18: Canti can recall relevant context via semantic similarity search (embeddings) without scanning full files
- FR19: RustyClaw can maintain dual-store consistency between source files and retrieval index
- FR20: RustyClaw can asynchronously index memory file changes into the retrieval store without blocking the active session

**Muscle Memory — Procedural / Habit Dojo (5 FRs)**
- FR21: RustyClaw can load Habit Dojo skills into the system prompt based on relevance
- FR22: Canti can query the skill registry for matching habits given a task context
- FR23: Canti can inspect a skill's provenance (origin, reasoning, what it replaced)
- FR24: Canti can install new skills via the Habit Dojo with full provenance metadata
- FR25: RustyClaw can position skills in cache-aware prompt zones for optimal cache hit rates

**Scheduled Operations (4 FRs)**
- FR26: RustyClaw can execute cron jobs on configured schedules (cron expressions, intervals, one-shot)
- FR27: RustyClaw can persist cron job state across restarts
- FR28: RustyClaw can yield scheduled work to interactive messages (priority queue)
- FR29: Jay can add, modify, or disable cron jobs via configuration

**Observability & Operations (5 FRs)**
- FR30: Jay can query a health endpoint for system status (uptime, queue depth, context usage, last message time)
- FR31: RustyClaw can emit structured JSON logs for all significant events
- FR32: RustyClaw can hot-reload configuration from file changes without restart
- FR33: RustyClaw can integrate with systemd watchdog (sd_notify)
- FR34: Jay can view operational metrics (cache hit rates, token usage, response latency)

**Prompt Architecture (3 FRs)**
- FR35: RustyClaw can assemble prompts in zone-based layout (Zone 1: static identity, Zone 2: skills, Zone 3: conversation)
- FR36: RustyClaw can track cache hit rates per zone to inform optimization
- FR37: RustyClaw can collect metrics that feed into future OPT self-tuning

### Non-Functional Requirements (28 total)

**Performance (6 NFRs)**
- NFR1: Processing overhead < 500ms (excluding Signal transport and LLM inference)
- NFR2: 500ms budget includes queue routing, DB lookups, prompt assembly, file reads, skill matching, response routing
- NFR3: System prompt assembly < 200ms
- NFR4: Embeddings recall < 500ms
- NFR5: Idle: < 50MB RSS, < 1% CPU. Active: leverage available cores for parallelism
- NFR6: Async indexing must not introduce perceptible latency

**Security (7 NFRs)**
- NFR7: Credential files 0600 permissions; degraded-not-dead on violation (declare to Claude for remediation)
- NFR8: Sender allowlist fail-closed across all channel adapters
- NFR9: No private data exposed via health endpoint or logs
- NFR10: Outbound message pre-send sanitizer (redact secrets before any channel dispatch)
- NFR11: Log output redacts sensitive values by default
- NFR12: Config files support env var references for secrets
- NFR13: Session content not persisted except through explicit AI-controlled memory writes

**Reliability (9 NFRs)**
- NFR14: Crash recovery without data loss (persisted UUID + memory files)
- NFR15: Pre-rotation flush guarantee (3 retries, alert on failure, never rotate without confirmation)
- NFR16: systemd watchdog restart within 30 seconds
- NFR17: Signal adapter reconnection with exponential backoff
- NFR18: Message queue with retry (Tokio channels, trait-abstracted for future Redis/pub-sub)
- NFR19: 99.9% uptime (three nines) via hot-reload, graceful error handling, watchdog
- NFR20: Config hot-reload validates before applying; invalid config = warning, prior config stays active
- NFR21: Core file changes (SOUL.md, USER.md, MEMORY.md, skills) detected and applied in real-time with validity checks
- NFR22: Self-generating default config on missing/corrupted config; service always boots

**Integration (6 NFRs)**
- NFR23: signal-cli via JSON-RPC; RustyClaw does not manage signal-cli lifecycle
- NFR24: Claude CLI via `--output-format json`; graceful handling of format changes
- NFR25: PostgreSQL connection pooling via sqlx
- NFR26: Atomic file writes (write-to-temp, rename)
- NFR27: systemd Type=notify with sd_notify
- NFR28: External dependency versions pinnable, tested on Chaos Monkey first

### Additional Requirements & Constraints

**From Executive Summary:**
- Type-driven correctness: enums encode valid state transitions, traits enforce behavioral contracts, invalid states unrepresentable
- Zero-overhead orchestration: Rust processing adds no perceptible latency to LLM inference path

**From Cognitive Architecture:**
- 4-layer recall model: Capture → Recognition → Association → Override
- Layers 1-2 + 4 = MVP; Layer 3 (GraphRAG) = Growth
- Dual-store pattern: source files (authoring) + retrieval index (recall), async sync
- Skill provenance: what/where/why/what-it-replaced

**From System Daemon Requirements:**
- Single binary, Linux only (x86_64), systemd-managed
- Runtime deps: signal-cli (Java), claude CLI (Node.js) — both external
- Deploy: copy binary + restart; rollback via `.prev` binary
- All changes validated on Chaos Monkey before production
- Target < 50MB RSS idle

**From Scoping:**
- POC (6 weeks) → MVP (12 weeks) → Growth (6 months)
- OpenClaw runs in parallel until MVP passes 1-week gate
- Two-person team: Jay (architect/reviewer) + Canti (builder via Claude Code sub-agents)

### PRD Completeness Assessment

**Strengths:**
- ✅ Clear product promise with measurable definition ("memory and muscle memory never disappear")
- ✅ 37 FRs organized by capability area, implementation-agnostic
- ✅ 28 NFRs all specific and measurable (latency budgets, uptime targets, permission modes)
- ✅ 5 user journeys covering happy path, edge cases, ops, AI perspective, and background work
- ✅ POC → MVP → Growth phasing with explicit success gates
- ✅ Innovation analysis with validation approach and fallback plans
- ✅ Cognitive architecture well-defined with clear phase mapping per layer
- ✅ Design principles include enum+trait semantic correctness

**Gaps Identified:**
- ⚠️ Architecture document is pre-ceremony — needs rebuild to align with PRD
- ⚠️ Epics document is pre-ceremony — needs rebuild with POC→MVP→Growth phasing and FR traceability
- ⚠️ No FR explicitly covers reactions (Signal emoji reactions mentioned in Journey 1 context but no FR)
- ⚠️ No FR covers media/attachment handling via Signal (may be out of scope for MVP — clarify)
- ⚠️ Technical Success table shows "Message round-trip ≤ 10s" but NFR1 says "< 500ms excluding Signal and LLM" — these measure different things (both valid, but could confuse downstream readers)
- ⚠️ No explicit FR for the self-generating default config (NFR22) — NFRs shouldn't introduce new capabilities not traceable to FRs

## Epic Coverage Validation

**Note:** The epics document (`docs/epics/epics.md`) predates the PRD ceremony. It uses a different FR numbering system (18 FRs vs finalized 37) and a flat phasing model (no POC→MVP→Growth distinction). Coverage is mapped by capability, not by number.

### Coverage Matrix

| PRD FR | Capability | Epic Coverage | Status |
|--------|-----------|---------------|--------|
| FR1 | Signal send/receive | Epic 1: Story 1.6 | ✅ Covered |
| FR2 | Typing indicator | Epic 1: Story 1.6 | ✅ Covered |
| FR3 | Queue when busy | Epic 1: Story 1.5 | ✅ Covered |
| FR4 | Priority interactive > cron | Epic 1: Story 1.5 | ✅ Covered |
| FR5 | Sender allowlist auth | Epic 1: Story 1.6 | ✅ Covered |
| FR6 | Reject unauthorized silently | Epic 1: Story 1.6 | ✅ Covered |
| FR7 | Create new session | Epic 1: Story 1.7, 1.8 | ✅ Covered |
| FR8 | Resume session by UUID | Epic 1: Story 1.7, 1.8 | ✅ Covered |
| FR9 | Persist UUID to disk | Epic 1: Story 1.8 | ✅ Covered |
| FR10 | Monitor context usage | Epic 2: Story 2.1 | ✅ Covered |
| FR11 | Soft threshold injection | Epic 2: Story 2.5 | ✅ Covered |
| FR12 | Pre-rotation flush guarantee | Epic 2: Story 2.2 | ✅ Covered |
| FR13 | Rotate with summary injection | Epic 2: Story 2.3 | ✅ Covered |
| FR14 | Daily scheduled rotation | Epic 2: Story 2.4 | ✅ Covered |
| FR15 | Flush on SIGTERM | **NOT FOUND** | ❌ MISSING |
| FR16 | Prompt assembly from files | Epic 1: Story 1.9 | ✅ Covered |
| FR17 | Index files into pgvector | **NOT FOUND** | ❌ MISSING |
| FR18 | Semantic recall via embeddings | Epic 5: Story 5.1 (MCP) | ⚠️ Partial (MCP only, not core) |
| FR19 | Dual-store consistency | **NOT FOUND** | ❌ MISSING |
| FR20 | Async indexing without blocking | **NOT FOUND** | ❌ MISSING |
| FR21 | Load Habit Dojo skills into prompt | Epic 4: Story 4.2 | ⚠️ Partial (skill loading, not Habit Dojo integration) |
| FR22 | Query skill registry by context | Epic 5: Story 5.2 (MCP) | ⚠️ Partial (MCP only) |
| FR23 | Inspect skill provenance | **NOT FOUND** | ❌ MISSING |
| FR24 | Install new skills with provenance | **NOT FOUND** | ❌ MISSING |
| FR25 | Cache-aware skill positioning | Epic 4: Story 4.1 | ✅ Covered |
| FR26 | Execute cron jobs | Epic 3: Story 3.1 | ✅ Covered |
| FR27 | Persist cron state across restarts | Epic 3: Story 3.2 | ✅ Covered |
| FR28 | Yield cron to interactive | Epic 3: Story 3.1 | ✅ Covered |
| FR29 | Add/modify/disable cron via config | Epic 3: Story 3.1 | ✅ Covered |
| FR30 | Health endpoint | Epic 3: Story 3.6 | ✅ Covered |
| FR31 | Structured JSON logs | Epic 3: Story 3.4 | ✅ Covered |
| FR32 | Config hot-reload | Epic 4: Story 4.4 | ✅ Covered |
| FR33 | systemd watchdog | **NOT FOUND** | ❌ MISSING |
| FR34 | View operational metrics | Epic 3: Story 3.5 | ✅ Covered |
| FR35 | Zone-based prompt layout | Epic 4: Story 4.1 | ✅ Covered |
| FR36 | Track cache hits per zone | Epic 3: Story 3.5 | ⚠️ Partial (metrics generic, not zone-specific) |
| FR37 | Collect metrics for OPT | Epic 3: Story 3.5 | ✅ Covered |

### Coverage Statistics

- **Total PRD FRs:** 37
- **Fully covered:** 23 (62%)
- **Partially covered:** 4 (11%)
- **Missing:** 10 (27%)

### Missing Requirements (Critical)

**FR15: Graceful shutdown with memory flush on SIGTERM**
- Impact: Data loss risk on restart/deploy
- Recommendation: Add to Epic 1 (core lifecycle) or Epic 2 (session lifecycle)

**FR17: Index memory files into pgvector as files change**
- Impact: Layer 2 (Recognition) doesn't exist without this
- Recommendation: New epic or add to MVP-phase stories

**FR19: Dual-store consistency between source files and retrieval index**
- Impact: Stale embeddings = wrong recall results
- Recommendation: Pair with FR17 in memory persistence epic

**FR20: Async indexing without blocking active session**
- Impact: Indexing could degrade interactive performance
- Recommendation: Pair with FR17/FR19

**FR23: Inspect skill provenance**
- Impact: Skill introspection impossible — "where did this habit come from?" unanswerable
- Recommendation: New Habit Dojo epic or extend Epic 5

**FR24: Install new skills with provenance metadata**
- Impact: Habit Dojo can't grow — no new skill installation path
- Recommendation: Pair with FR23

**FR33: systemd watchdog integration**
- Impact: Hangs go undetected — contradicts NFR16 (30s restart on hang)
- Recommendation: Add to Epic 1 (daemon lifecycle)

### Missing Requirements (High Priority)

**FR18 (partial): Semantic recall as core capability, not just MCP**
- Epics treat memory search as an MCP server (Epic 5). PRD positions embeddings recall as MVP-critical (Layer 2). Need to clarify: does Claude use MCP for recall, or does RustyClaw inject recall results into prompt assembly?

**FR21 (partial): Habit Dojo integration vs generic skill loading**
- Epics cover skill loading mechanics (Epic 4) but don't reference Habit Dojo, provenance, or the skill registry. The *mechanism* exists but the *integration* is missing.

**FR36 (partial): Zone-specific cache tracking**
- Epics collect generic metrics but don't break down by prompt zone. Zone-level granularity is needed for OPT to optimize zone boundaries.

### Structural Issues

1. **No POC→MVP→Growth phasing in epics.** Epics use a flat 6-epic structure. PRD defines explicit phase gates. Epics need to map stories to phases.
2. **No FR traceability.** Epics reference their own 18-FR list, not the finalized 37 FRs. The FR coverage map is stale.
3. **Memory persistence has no epic.** FR17-FR20 (embeddings, dual-store, async indexing) have no home. This is the Layer 2 (Recognition) gap.
4. **Habit Dojo has no epic.** FR21-FR24 (skill loading with provenance, registry queries, installation) are scattered across Epic 4 and Epic 5 without cohesion.
5. **OPT is in MVP epics but PRD scopes it to Growth.** Epic 4 includes OPT (Story 4.3) but PRD explicitly defers OPT to Growth phase.

## UX Alignment

**N/A** — RustyClaw is a headless system daemon with no GUI. The user interface is Signal (managed by signal-cli, not by RustyClaw). No UX document required. The health endpoint (FR30) returns JSON for programmatic consumption, not human interaction.

## Epic Quality Review

**Note:** This reviews the pre-ceremony epics (`docs/epics/epics.md`). These will be rebuilt; this review captures what to fix.

### Epic Structure Validation

| Epic | User Value? | Independent? | Verdict |
|------|------------|-------------|---------|
| Epic 1: Signal Bridge MVP | ✅ Yes — "Jay texts, Canti responds" | ✅ Standalone | Good |
| Epic 2: Session Lifecycle | ✅ Yes — "conversation never dies" | ✅ Depends on E1 only | Good |
| Epic 3: Scheduling & Observability | ⚠️ Mixed — cron is user value, logging is technical | ✅ Depends on E1-2 | Split recommended |
| Epic 4: Prompt Optimization & OPT | 🔴 Technical — "cache-efficient prompts" is infrastructure | ⚠️ Depends on E3 metrics | Reframe as user value |
| Epic 5: MCP Servers | ⚠️ Mixed — memory search is user value, lifecycle is technical | ✅ Parallel with E1+ | Split recommended |
| Epic 6: Decommission OpenClaw | ✅ Yes — "only one system running" | ✅ Depends on all | Good |

### 🔴 Critical Violations

1. **No POC→MVP→Growth phasing.** Stories are ordered by epic, not by phase gate. A developer reading this can't tell what to build first for POC vs what waits for MVP.

2. **Epic 4 is a technical milestone.** "Prompt Optimization & OPT" bundles cache mechanics with self-tuning. User value is invisible. Reframe: "Canti remembers habits and learns from experience" (skill loading + OPT).

3. **FR coverage gaps.** 10 of 37 FRs have no epic home. Memory persistence (FR17-20) and Habit Dojo (FR21-24) are architecturally significant and have zero coverage.

4. **OPT scoped to wrong phase.** Epic 4 includes OPT (Story 4.3) but PRD scopes OPT to Growth. Building it in MVP steals effort from higher-priority muscle memory work.

### 🟠 Major Issues

5. **Story 1.2 crashes on invalid config.** Acceptance criteria says "fails with a descriptive error and does not proceed." PRD (NFR22) says generate defaults and start degraded. Contradicts the degraded-not-dead principle.

6. **No SIGTERM flush story.** FR15 (graceful shutdown) has no story. This is a data loss risk on every deploy.

7. **No systemd watchdog story.** FR33/NFR16 require sd_notify integration. No story covers this.

8. **Epic 3 bundles scheduling + observability.** These are separate concerns. Cron scheduler is user-facing; structured logging is ops infrastructure. Split for cleaner independence.

9. **Stale FR numbering.** Epics reference 18 FRs; PRD defines 37. The FR Coverage Map is completely stale.

### 🟡 Minor Concerns

10. **Story quality is generally good.** Given/When/Then format used consistently. Acceptance criteria are testable. Stories are mostly atomic.

11. **Story 1.1 (scaffolding) is fine for brownfield.** CI, clippy, rustfmt — proper foundation.

12. **Estimated effort is reasonable.** 10-15 weeks aligns with PRD's 12-week MVP timeline.

### Story Dependency Analysis

Within-epic dependencies are clean — stories build on prior stories within their epic without forward references. Cross-epic dependencies follow the correct direction (E2 depends on E1, E3 depends on E2). No circular dependencies detected.

### Recommendation

**The epics document needs to be rebuilt** via the BMAD create-epics workflow using the finalized PRD as input. Key changes:
- Restructure into POC / MVP / Growth phases
- Add Memory Persistence epic (FR17-FR20)
- Add Habit Dojo epic (FR21-FR24)
- Move OPT from MVP to Growth
- Add SIGTERM flush and systemd watchdog to core lifecycle
- Establish FR traceability from every story back to PRD FRs

## Summary and Recommendations

### Overall Readiness Status

**🟡 NEEDS WORK** — PRD is strong. Architecture and Epics need rebuilding.

### Scorecard

| Artifact | Status | Assessment |
|----------|--------|------------|
| **PRD** | ✅ Ready | 37 FRs, 28 NFRs, complete cognitive architecture, clear phasing. Minor gaps (3 missing FRs for reactions, media, default config generation). |
| **Architecture** | 🔴 Not Ready | Pre-ceremony draft. Does not reflect Memory/Muscle Memory, cognitive layers, dual-store, skill provenance, or POC→MVP→Growth phasing. |
| **Epics & Stories** | 🔴 Not Ready | Pre-ceremony. 27% FR coverage gap (10 of 37 FRs missing). No phase alignment. OPT in wrong phase. Story 1.2 contradicts NFR22. |
| **UX Design** | ✅ N/A | Headless daemon. No UX required. |

### Critical Issues Requiring Immediate Action

1. **Architecture document must be rebuilt.** The PRD defines a cognitive architecture (4 layers, dual-store, skill provenance, enum+trait semantic correctness) that the architecture doc doesn't reflect. Every downstream decision (module structure, trait definitions, data flows) depends on this.

2. **Epics must be rebuilt with FR traceability.** 10 FRs have no epic home. Memory persistence and Habit Dojo — two of the three things that make MVP different from POC — have zero story coverage.

3. **PRD minor gaps to close:**
   - Add FR for self-generating default config (currently only in NFR22 — capability without FR)
   - Clarify Signal reactions and media handling scope (in or out of MVP?)
   - Align "10s round-trip" metric with "500ms overhead" NFR (both valid, add clarifying note)

### Recommended Next Steps

1. **Close PRD gaps** (30 min) — Add 1-2 missing FRs, clarify scope questions
2. **Run BMAD create-architecture** — Rebuild architecture doc from finalized PRD. Define traits, modules, data flows aligned with cognitive architecture.
3. **Run BMAD create-epics** — Rebuild epics from finalized PRD + architecture. Map every FR to a story. Phase-gate every story into POC/MVP/Growth.
4. **Re-run readiness check** — Validate coverage after rebuild

### Final Note

This assessment identified **16 issues** across **4 categories** (PRD gaps, coverage gaps, structural violations, quality concerns). The PRD is the strongest artifact — it captures a sophisticated product vision with measurable requirements. The architecture and epics documents are pre-ceremony drafts that served their purpose as input to the PRD but are now stale. Rebuilding them using the finalized PRD as the authoritative source will close all critical gaps.

**Assessed by:** Canti (Lord Canti) — March 2, 2026
