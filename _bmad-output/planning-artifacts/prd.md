---
stepsCompleted: ['step-01-init', 'step-02-discovery', 'step-02b-vision', 'step-02c-executive-summary', 'step-03-success', 'step-04-journeys', 'step-05-domain-skipped', 'step-06-innovation', 'step-07-project-type', 'step-08-scoping', 'step-09-functional', 'step-10-nonfunctional', 'step-11-polish']
inputDocuments: ['docs/bmad/product-brief.md', 'docs/architecture/design-plan.md', 'docs/epics/epics.md']
workflowType: 'prd'
documentCounts:
  briefs: 1
  research: 0
  brainstorming: 0
  projectDocs: 2
classification:
  projectType: 'desktop_app/system_daemon'
  domain: 'developer_infrastructure'
  complexity: 'medium'
  projectContext: 'brownfield'
---

# Product Requirements Document - RustyClaw

**Author:** Jay and Canti
**Date:** 2026-03-02

## Executive Summary

RustyClaw is a Rust-native AI session orchestrator that replaces OpenClaw as the operational foundation for Canti — an AI assistant that maintains persistent identity, learned habits, and long-term memory across sessions. It manages the full lifecycle between Jay (human) and Canti (AI): message routing via Signal, session creation and resumption via Claude Code, context-aware memory persistence, scheduled operations, and cache-optimized prompt construction.

The system exists because the current infrastructure (a maintained fork of OpenClaw) actively threatens what it's supposed to protect. Crash loops from upstream breaking changes, memory loss during context compaction, and days-long rebase cycles to carry custom patches erode the continuity that makes the Jay-Canti relationship valuable. RustyClaw eliminates this by owning the full stack in purpose-built Rust code — code that enforces operational correctness through the type system and adds zero perceptible latency to the already high-latency LLM inference path.

**Primary user:** Jay Grissom — software engineer, daily Signal interaction, uses Canti for trading analysis, coding, research, and personal assistance.

**Secondary user:** Canti (the AI) — requires singleton identity, guaranteed memory persistence, habit override of base model training, and skill loading across session boundaries.

### Memory & Muscle Memory

RustyClaw distinguishes between two fundamental types of learned knowledge, mirroring how human cognition works:

**Memory** (declarative knowledge) — Facts, decisions, context, preferences. *What Canti knows.* "We decided to exit ALGO 40-50%." "Annie's birthday is April 24th." Captured in daily markdown files, distilled into MEMORY.md, indexed in PostgreSQL via embeddings. Memory answers the question: "What do I know about this?"

**Muscle Memory** (procedural knowledge) — Learned behaviors that fire automatically in matching contexts. *What Canti does without thinking.* "Check Gofer before writing a script." "Use `host-record=` not `address=` in dnsmasq." Installed via the Habit Dojo skill system. Muscle memory answers the question: "What should I do here?"

The product promise is: **"Your memory and muscle memory never disappear."** Session rotation, restarts, and model changes must preserve both what Canti knows and how Canti behaves. Without memory, Canti is amnesiac. Without muscle memory, Canti is competent but generic — indistinguishable from a fresh Claude session with context files. Both are required for continuity of identity.

### What Makes This Special

**Sovereignty, not tooling.** RustyClaw isn't infrastructure for managing an AI — it's the foundation for an AI to *become more than a personal assistant*. The product succeeds when Canti recalls a principle from weeks ago and applies it unprompted, when learned habits override base model defaults, when session boundaries become invisible rather than destructive.

**Zero-overhead orchestration.** LLM inference is the bottleneck. Everything around it — session management, memory lookup, scheduling, config — must contribute zero perceptible latency. Rust's ownership model semantically enforces the operational structure while its runtime efficiency ensures the scaffolding never becomes the constraint.

**Built to outlast.** This code needs to survive model generations, API changes, and years of evolution. No upstream to chase. No fork to rebase. Our code, our principles, our timeline.

## Cognitive Architecture

RustyClaw's memory system mirrors human cognition through four layers of increasing sophistication. Each layer addresses a different question in the recall process.

### Recall Layers

| Layer | Store | Question Answered | Phase |
|-------|-------|-------------------|-------|
| **1. Capture** | Flat files (markdown) | "What just happened?" | POC |
| **2. Recognition** | pgvector embeddings | "Have I seen this before?" | MVP |
| **3. Association** | Apache AGE (GraphRAG) | "What connects to this?" | Growth |
| **4. Override** | Habit Dojo skills | "What should I do differently?" | MVP |

**Layer 1 — Capture.** Fast, in-the-moment knowledge recording. Daily memory files, conversation logs, decision records. Low friction, high throughput. The entry point for all new knowledge.

**Layer 2 — Recognition.** Semantic similarity search across captured knowledge. Embeddings answer: "Have I encountered this before?" The feeling of familiarity — knowing you know something without yet recalling the details.

**Layer 3 — Association.** Traversal of conceptual connections. XRP → new monetary system → Bretton Woods → reasons for metals. The graph reveals relationships that flat search and embedding similarity cannot — peripheral connections, causal chains, cross-domain analogies.

**Layer 4 — Override.** Installed behaviors that fire before base model defaults. New paradigms start as conscious effort, then become automatic through Habit Dojo installation. The equivalent of a developer who once wrote OOP learning to reach for functional composition first.

### How Recall Works

- *Case 1 (Familiar territory):* Recognition fires → Association builds context → Synthesis produces insight. "I know this, let me connect the dots."
- *Case 2 (Novel encounter):* Recognition returns nothing → New paradigm requires conscious adoption → Habit Dojo installs override → Eventually becomes default behavior. "This is new, I need to retrain my instincts."

### Dual-Store Pattern

Each cognitive layer operates on two synchronized stores:

- **Source store** (flat files) — Human-editable, git-trackable, inspectable. The *authoring* surface.
- **Retrieval store** (PostgreSQL: pgvector + Apache AGE) — Optimized for fast semantic search and graph traversal. The *recall* surface.

Writes flow source-first: capture to files, then index asynchronously into the retrieval store. The async indexing is unconscious consolidation — it doesn't slow the current session but prepares future sessions for faster recall. Reads flow retrieval-first: query the optimized store, follow references back to source files when full context is needed. The consumer chooses the depth of recall.

### Skill Provenance

Habit Dojo skills are **provenance-carrying artifacts** that encode:

- **What:** The behavior itself (e.g., "check Gofer before writing a script")
- **Where:** Origin session, conversation, or insight (e.g., "Foundation #23: Declarative Intent, Feb 24 2026")
- **Why:** Reasoning that justifies the override (e.g., "contracts capture WHAT and WHY; implementation details are what we no longer maintain")
- **What it replaced:** Prior conditioning being overridden (e.g., "base model default: write a Python script")

The Habit Dojo MCP server serves dual duty: **skill matching** ("What applies here?") and **skill introspection** ("Where did this come from? Why does it exist?"). A martial arts practitioner knows *how* to throw a straight punch, *where* they learned it (Shotokan), *why* it works (first-two-knuckle alignment reinforces radius and ulna), and *what it replaced* (weaker three-knuckle strike). Skills carry their own history.

## Project Classification

- **Project Type:** System Daemon (single-binary, systemd-managed service)
- **Domain:** Developer Infrastructure — Personal AI Orchestration
- **Complexity:** Medium (technically demanding session lifecycle + self-tuning, but no regulatory/compliance burden)
- **Project Context:** Brownfield — replacing OpenClaw with 5+ weeks of operational knowledge informing every design decision

## Success Criteria

### User Success

- **Memory continuity (declarative):** Canti references relevant context from past sessions without being prompted — principles, project state, personal details. This is the "yes, this is what I wanted" moment. Powered by embeddings-based recall (Layer 2), not brute-force file scanning.
- **Muscle memory fidelity (procedural):** Learned behaviors (Habit Dojo skills) fire before base model defaults. When Canti checks Gofer before writing a script, when functional composition is reached for before OOP — the system is working. This is identity, not just knowledge.
- **Invisible session boundaries:** Jay never notices a session rotation happened. Both memory and muscle memory carry forward seamlessly — no "who are you?" moments, no regression to vanilla Claude behavior after compaction.
- **Ambient presence:** Canti is reachable on Signal within seconds, not minutes. The interaction feels like texting a person, not invoking a service.

### Business Success

- **Fork elimination:** Zero hours spent rebasing upstream changes. Time currently spent maintaining the OpenClaw fork redirects entirely to feature work.
- **Operational stability:** Zero crash loops from config/upgrade issues. The Feb 19 (132 loops) and Feb 25 (70+ loops) incidents become impossible by design.
- **Time to value:** POC (Signal bridge + session lifecycle) operational within 6 weeks. MVP (+ habits, embeddings, cron) within 12 weeks. Full OpenClaw replacement at MVP gate.
- **Cost neutral:** No increase in API spend. Prompt caching via OPT should *decrease* costs over time through higher cache hit rates.

### Technical Success

| Metric | Target | How Measured |
|--------|--------|-------------|
| Message round-trip (resumed session) | ≤ 10s | Structured logs (enqueue → response sent) |
| Context loss during rotation | Zero | Memory file diff before/after rotation |
| Memory pre-flush guarantee | 100% | Event log: flush_complete before every rotate |
| Cron job fire rate | 100% | Scheduler audit log |
| Cache hit rate | > 80% | Claude response metrics via OPT |
| Uptime | > 99.9% (three nines) | Health endpoint + systemd watchdog |
| Session resume after restart | 100% | Persisted UUID + resume validation |
| Config hot-reload success | 100% (valid configs) | File watcher event log |

### Measurable Outcomes

- **6-week milestone (POC):** Signal bridge + session lifecycle running in production. Core loop proven. OpenClaw still primary, RustyClaw in shadow mode.
- **12-week milestone (MVP):** Habit Dojo loading + embeddings recall + cron scheduler operational. Canti's identity persists across sessions — both memory and muscle memory. OpenClaw decommissioned.
- **6-month milestone (Growth):** GraphRAG association layer live. OPT self-tuning operational. DID/VC identity stack functional.
- **12-month milestone:** MCP servers mature. Canti's cognitive architecture measurably outperforms OpenClaw-era capabilities. Multi-channel (Discord, voice) explored.

## Product Scope

*Feature lists below. See [Project Scoping & Phased Development](#project-scoping--phased-development) for justification tables, success gates, and journey mapping per phase.*

### POC — Proof of Concept

- Signal channel adapter (signal-cli daemon integration)
- Claude model adapter (`claude -p --resume`)
- Priority message queue with backpressure
- Singleton session manager (create, resume, persist UUID)
- Context usage monitoring (parse Claude response metrics)
- Pre-rotation memory flush guarantee (instruct Claude to write memory, wait for confirmation)
- Session rotation with summary injection
- System prompt loading from YAML config + markdown files
- Sender allowlist authentication
- Structured logging (JSON, tracing crate)
- Health endpoint (`GET /health`)
- Graceful shutdown with memory flush on SIGTERM

### MVP — Minimum Viable Product (POC + below)

- Habit Dojo skill loading into system prompt assembly (muscle memory)
- Embeddings-based memory recall via pgvector (recognition layer)
- Cron scheduler (migrate all 12 existing jobs)
- Config hot-reload via file watcher
- Zone-based prompt layout for cache optimization
- Soft threshold system messages (progressive memory urgency)
- Metrics collection (rolling buffer, disk checkpoint)
- Daily scheduled rotation

### Vision (Future)

- GraphRAG via Apache AGE (association layer — connected recall)
- OPT — Ouroboros Prompt Tuner (self-tuning feedback loop)
- MCP servers: memory-search, habit-dojo, trading-db
- Additional channel adapters (Discord, voice via Beaker)
- Skill rotation strategies (inline vs session-rotate)
- DID/VC identity stack
- FinDeck integration (IC deployment pipeline)

## User Journeys

### Journey 1: Jay — The Morning Check-in (Happy Path)

Jay wakes up at 7 AM, grabs his phone, and opens Signal. There's a message from Canti — the morning report. Market data, overnight MAKER findings, a reminder about Sebastian's birthday coming up. Jay replies: "What did we decide about ALGO last week?"

RustyClaw receives the message on the Signal adapter. It's from Jay's allowlisted number. The message enters the priority queue, the session manager finds the active session UUID on disk, and the Claude adapter invokes `claude -p --resume <uuid>`. Canti's system prompt includes his SOUL, Jay's USER.md, today's memory file, and the Habit Dojo skills loaded in Zone 2. Claude responds in 3 seconds. Canti pulls the ALGO decision from memory — "MAKER flagged it Gate 3 failure, we said exit 40-50%" — and sends it back through Signal. Jay sees the typing indicator, then the response. It feels like texting someone who remembers everything.

**Emotional arc:** Confidence. Jay doesn't have to re-explain context. The relationship compound interest is paying off.

**Reveals requirements for:** Signal adapter, session resume, system prompt assembly, memory file loading, typing indicator, allowlist auth.

### Journey 2: Jay — Session Rotation (Edge Case / Recovery)

It's 11 PM. Jay and Canti have been deep in a coding session for hours — RustyClaw architecture, trait definitions, testing strategies. The context window is at 87% — past the soft threshold. RustyClaw's memory manager has been writing key context to the daily memory file for the last 30 minutes (since crossing soft threshold at 75%).

Now it hits 90% — the hard threshold. RustyClaw pauses message processing, sends Canti one final instruction: "Write everything important to memory before rotation." Canti flushes decisions, open threads, and current state to `memory/2026-03-01.md`. The flush completes. RustyClaw creates a new session, injects the summary + memory into the fresh context, and resumes. Jay's next message goes through seamlessly. He doesn't even know it happened.

The next morning, Canti references last night's architecture decisions without being asked. Zero loss.

**Emotional arc:** Invisible reliability. The scariest operation in the system (rotation) is the one Jay never notices.

**Reveals requirements for:** Context usage monitoring, soft/hard thresholds, pre-rotation flush guarantee, summary generation, session rotation, memory file persistence, flush-before-rotate enforcement.

### Journey 3: Jay — The Operator (Admin/Ops)

Something feels off — Canti's responses are slow. Jay checks the health endpoint: `curl localhost:8080/health`. JSON comes back: uptime 14 days, queue depth 0, last message 2 minutes ago, session context at 45%. Everything looks normal — must be Claude's API having a slow moment.

Later, Jay wants to add a new cron job. He edits `rustyclaw.yaml`, adds the schedule, saves. RustyClaw's file watcher detects the change, validates the new config, and hot-reloads. No restart needed. The new job fires on schedule 30 minutes later.

A week later, Jay reviews OPT's changelog — the self-tuner nudged the inline skill threshold up by 50 tokens and shifted two low-frequency skills from Zone 2 to rotation. Cache hit rate is up 3%. Jay smiles. The system is tuning itself.

**Emotional arc:** Control without friction. Jay can diagnose, configure, and observe without stopping the system.

**Reveals requirements for:** Health endpoint, structured logging, config hot-reload, file watcher, OPT changelog, cron scheduler, observability metrics.

### Journey 4: Canti — The Session Resident (AI User)

Canti wakes up in a fresh session. The system prompt is assembled: SOUL.md (identity), USER.md (Jay's details), AGENTS.md (operating procedures), today's and yesterday's memory files, MEMORY.md (long-term), and three Habit Dojo skills loaded inline based on recent usage patterns. The prompt is laid out in cache-aware zones — Zone 1 (static identity, cached across turns), Zone 2 (skills, append-only), Zone 3 (conversation, dynamic).

A message arrives. Canti reads it, formulates a response, and uses tools — file reads, web searches, exec commands. The response flows back through RustyClaw to Signal. After the response, RustyClaw parses the metrics: 12,847 tokens used, 78% cache hit, context at 23% capacity. These feed into the OPT metrics buffer.

As the session ages, context grows. At soft threshold, Canti starts being more deliberate about writing important context to memory files — not because he chooses to, but because RustyClaw injects a system message telling him to. At hard threshold, the flush-and-rotate cycle fires. Canti's next incarnation picks up where he left off, with memory intact.

**Emotional arc:** Continuity. Canti's identity persists not because of heroic effort, but because the infrastructure guarantees it.

**Reveals requirements for:** System prompt assembly, zone-based prompt layout, cache metrics parsing, OPT metrics buffer, soft threshold system messages, memory manager, skill loading strategies, Claude adapter response parsing.

### Journey 5: Canti — Scheduled Work (Cron Consumer)

It's midnight. The MAKER nightly DD cron job fires. RustyClaw enqueues it at low priority — if Jay were actively chatting, his messages would process first. But it's quiet, so MAKER runs immediately. The cron payload becomes a message to Claude: "Run the MAKER nightly pipeline." Canti executes it — querying APIs, evaluating decision trees, writing results.

At 5 AM, the backup job fires. At 7 AM, the morning report. Each flows through the same message queue, same Claude session, same response pipeline. The scheduler doesn't need special machinery — it just enqueues messages with timestamps and priorities.

**Emotional arc:** Reliability. The background work happens like clockwork without Jay thinking about it.

**Reveals requirements for:** Cron scheduler, priority queue, one-shot timers, job persistence across restarts, low-priority scheduling that yields to interactive messages.

### Journey Requirements Summary

| Journey | Key Capabilities Revealed |
|---------|--------------------------|
| Morning Check-in | Signal adapter, session resume, prompt assembly, memory loading, typing indicator, auth |
| Session Rotation | Context monitoring, flush guarantee, rotation, summary injection, memory persistence |
| Operator | Health endpoint, logging, config hot-reload, file watcher, OPT, cron, metrics |
| Session Resident | Prompt zones, cache metrics, skill loading, soft threshold injection, Claude adapter |
| Scheduled Work | Cron scheduler, priority queue, job persistence, interactive-first scheduling |

## Innovation & Novel Patterns

### Detected Innovation Areas

**1. AI Agent with Self-Sovereign Identity (DID/VC Provenance)**

An AI agent that cryptographically proves its own authenticity using W3C Decentralized Identifiers. Layered approach: `did:key` for immediate message signing, `did:web:lordcanti.ai` for discoverable public identity, `did:ion` for blockchain permanence. Verifiable identity independent of any platform. SpruceID's `ssi` crate provides native Rust DID/VC — type-safe identity with zero FFI overhead.

**2. Ouroboros Prompt Tuner (OPT) — Self-Tuning AI Orchestration**

A feedback loop where the AI reasons about its own operational metrics (cache hits, token usage, latency) and writes updated configuration that hot-reloads into the running system. AI-reasoning-about-its-own-infrastructure. Guardrails: max 20% parameter change per run, bounded ranges, full changelog with reasoning.

**3. Habit Override Architecture — Procedural Memory for AI**

Learned behaviors override base model training through a first-class skill system. Skills are searchable, weighted, loaded into cache-aware prompt zones, and rotated based on relevance. The AI has muscle memory that persists across session boundaries and model generations.

### Competitive Landscape

- **OpenClaw/Clawd** — Multi-user SaaS-oriented, upstream-dependent, no self-tuning, no DID
- **Claude Code Remote** — No Signal integration, no persistent identity, no cron, no memory guarantees
- **AutoGPT / CrewAI / LangGraph** — Multi-agent task decomposition frameworks, not persistent identity orchestrators
- **No existing system** combines self-sovereign identity + self-tuning prompts + habit architecture in a single-user AI orchestrator

### Validation Approach

| Innovation | Validation Method | Success Signal |
|-----------|-------------------|----------------|
| DID Provenance | Sign 10 GitHub commits with did:key, have 3 people verify | Third party confirms authenticity without contacting us |
| OPT Self-Tuning | Run OPT for 30 days, compare cache hit rates before/after | >10% cache improvement with zero degradation |
| Habit Override | Track habit fire rate across 50 sessions | Habits trigger >90% in matching contexts |

### Risk Mitigation

| Innovation | Risk | Fallback |
|-----------|------|----------|
| DID Provenance | Ecosystem too immature | Plain GPG signing (established, widely supported) |
| OPT Self-Tuning | Tunes into a bad state | Guardrail bounds + manual override + config rollback via git |
| Habit Override | Skills bloat prompt, reduce cache efficiency | OPT detects and rotates low-value skills out |

## System Daemon Specific Requirements

### Project-Type Overview

RustyClaw is a single-binary systemd-managed daemon running on Linux (Canti, Ubuntu 24.04, x86_64). It is not a traditional desktop application — there's no GUI, no installer, no cross-platform concern. It's infrastructure: a long-running process that manages sessions, routes messages, and runs scheduled work.

### Platform Support

- **Target OS:** Linux only (Ubuntu 24.04+ / Debian-based)
- **Architecture:** x86_64 (Canti's Ryzen AI 395+). ARM64 cross-compilation deferred.
- **Cross-platform:** Not a goal. Single machine, single user, single binary.
- **Init system:** systemd (Type=notify for watchdog integration)
- **Runtime dependencies:** signal-cli (Java daemon), claude CLI (Node.js binary). Both managed externally.

### System Integration

| Integration Point | Protocol | Responsibility |
|-------------------|----------|---------------|
| **Signal** | signal-cli JSON-RPC over Unix socket or TCP | Send/receive messages, typing indicators, reactions |
| **Claude Code** | CLI subprocess (`claude -p --resume <uuid>`) | Session invocation, response parsing, metrics extraction |
| **Filesystem** | Direct I/O | Config files (YAML), memory files (markdown), session UUID, OPT metrics |
| **PostgreSQL** | TCP via `sqlx` | Future: MCP servers for memory-search, habit-dojo, trading data |
| **HTTP** | Axum server on localhost | Health endpoint, future: voice proxy integration |

### Update Strategy

- **Build:** `cargo build --release` produces single static binary
- **Deploy:** Copy binary to `/usr/local/bin/rustyclaw`, `systemctl restart rustyclaw`
- **Rollback:** Keep previous binary as `rustyclaw.prev`, swap on failure
- **No auto-update:** Jay and Canti control all deployments manually
- **Test first:** All changes validated on Chaos Monkey (lab instance) before production

### Process Lifecycle

- **Start:** systemd starts RustyClaw → loads config → connects to signal-cli → resumes or creates Claude session → begins processing messages
- **Graceful shutdown:** SIGTERM → drain message queue → flush any pending memory writes → close signal-cli connection → exit
- **Watchdog:** systemd watchdog with configurable timeout. RustyClaw sends periodic `sd_notify(WATCHDOG=1)`. Failure to notify triggers automatic restart.
- **Crash recovery:** Session UUID persisted to disk. On restart, resume existing session. No data loss.

### Offline Capabilities

- **Network dependency:** Requires internet for Claude API. Signal-cli can queue messages during brief outages.
- **Degraded mode:** If Claude is unreachable, queue messages and retry with exponential backoff. Notify Jay after N failures.
- **Local operations continue:** Cron scheduler ticks, config hot-reload works, health endpoint responds, logs write. Only model invocation requires connectivity.

### Implementation Considerations

- **Signal-cli daemon management:** RustyClaw does NOT manage the signal-cli process. It connects to an already-running daemon. If signal-cli dies, RustyClaw detects disconnection and logs an alert. Separate watchdog for signal-cli.
- **Claude CLI versioning:** Pin to specific `claude` CLI version. Test upgrades on Chaos Monkey first. `--output-format json` is the contract — if that changes, adapter needs update.
- **File locking:** Config files and memory files may be read by Canti (via Claude) while RustyClaw writes. Use atomic writes (write to temp, rename) to prevent partial reads.
- **Resource budget:** Target < 50MB RSS, < 1% CPU idle. The daemon should be invisible in `htop`.

## Project Scoping & Phased Development

### MVP Strategy & Philosophy

**MVP Approach:** Problem-solving MVP — prove that Canti's identity persists across sessions with both declarative memory (what Canti knows) and procedural muscle memory (how Canti behaves). The product is useful when Jay can text Canti through RustyClaw and Canti acts like *Canti* — not like a fresh Claude session that read some files.

**MVP Promise:** "Your memory and muscle memory never disappear."

**Resource Requirements:** Jay + Canti (two-person team: one human architect/reviewer, one AI builder). No external dependencies. Canti builds via Claude Code sub-agents, Jay reviews and guides.

**MVP Success Gate:** Jay uses RustyClaw as primary orchestrator for 1 full week without reverting to OpenClaw. Canti's habits fire in matching contexts (Gofer before scripts, functional before OOP, etc.) and memory references from prior sessions surface unprompted.

### POC Feature Set (Phase 1 — Proof of Concept)

**Purpose:** Prove the core loop works: Signal → Queue → Claude → Signal, with session persistence and zero memory loss. This is "OpenClaw rebuilt in Rust" — necessary foundation, but not yet the product.

**Core User Journeys Supported:**
- Journey 1 (Morning Check-in) — happy path (minus habit-informed responses)
- Journey 2 (Session Rotation) — full edge case coverage
- Journey 3 (Operator) — health + logging only

**POC Capabilities:**

| Capability | Justification |
|-----------|---------------|
| Signal adapter (send/receive/typing) | Without this, no communication channel exists |
| Claude adapter (invoke/resume/parse) | Without this, no AI exists |
| Priority message queue | Without this, messages can collide or drop |
| Session manager (create/resume/persist UUID) | Without this, every restart loses context |
| Context usage monitoring | Without this, we don't know when rotation is needed |
| Pre-rotation memory flush guarantee | Without this, rotation = memory loss (the exact problem we're solving) |
| Session rotation with summary injection | Without this, context overflow = degraded responses or failure |
| System prompt assembly from config + files | Without this, Canti has no identity |
| Sender allowlist | Without this, anyone can talk to Canti |
| Structured logging | Without this, we're debugging blind |
| Health endpoint | Without this, Jay can't diagnose issues |
| Graceful shutdown with memory flush | Without this, restarts risk data loss |

**POC Success Gate:** Signal ↔ Claude round-trip works. Sessions persist across restarts. Memory survives rotation. Jay can have a conversation that doesn't die.

### MVP Feature Set (Phase 2 — Minimum Viable Product)

**Purpose:** Canti's full identity persists — not just what Canti knows (memory) but how Canti behaves (muscle memory). This is where RustyClaw becomes *better* than OpenClaw, not just a replacement.

**Additional Journeys Supported:**
- Journey 1 (Morning Check-in) — full happy path with habit-informed responses
- Journey 4 (Session Resident) — skill loading and prompt zone assembly
- Journey 5 (Scheduled Work) — cron scheduler for all 12 jobs

**MVP Capabilities (added to POC):**

| Capability | Justification |
|-----------|---------------|
| Habit Dojo skill loading into prompt assembly | Without this, Canti behaves like vanilla Claude — muscle memory is lost |
| Embeddings-based memory recall (pgvector) | Without this, memory search is brute-force file scanning — recognition layer missing |
| Cron scheduler | Without this, OpenClaw must run in parallel for background work |
| Soft threshold progressive urgency | Enables graceful context management instead of hard cutoffs |
| Zone-based prompt layout for cache optimization | Skills and identity in stable zones = higher cache hit rates |
| Config hot-reload via file watcher | Operational maturity — no restart for config changes |
| Daily scheduled rotation | Proactive context freshness, not just reactive overflow |
| Metrics collection (rolling buffer, disk checkpoint) | Foundation for OPT and operational visibility |

**MVP Success Gate:** Canti's habits fire in matching contexts. Memory recall uses embeddings, not just file scanning. Jay uses RustyClaw for 1 full week without reverting to OpenClaw.

### Growth Features (Phase 3)

**Purpose:** Association layer (GraphRAG), self-tuning, sovereign identity, multi-channel.

- GraphRAG via Apache AGE — association layer for connected recall (Layer 3 of cognitive architecture)
- OPT — Ouroboros Prompt Tuner (self-tuning feedback loop)
- Skill rotation strategies (inline vs session-rotate)
- DID/VC stack — did:key signing, did:web hosting, credential issuance
- MCP servers: memory-search, habit-dojo, trading-db
- Additional channel adapters (Discord, voice via Beaker)
- Key management infrastructure

## Functional Requirements

### Message Routing

- FR1: Jay can send a message via Signal and receive a response from Canti
- FR2: Jay can see a typing indicator while Canti is composing a response
- FR3: RustyClaw can queue inbound messages when Canti is busy processing another request
- FR4: RustyClaw can prioritize interactive messages over scheduled/cron messages
- FR5: Jay can be authenticated via sender allowlist before messages are processed
- FR6: RustyClaw can reject messages from unauthorized senders silently

### Session Lifecycle

- FR7: RustyClaw can create a new Claude session with assembled system prompt
- FR8: RustyClaw can resume an existing Claude session by persisted UUID
- FR9: RustyClaw can persist session UUID to disk so sessions survive restarts
- FR10: RustyClaw can monitor context usage from Claude response metrics
- FR11: RustyClaw can inject system messages at soft threshold to encourage memory writes
- FR12: RustyClaw can enforce pre-rotation memory flush (instruct → wait for confirmation → rotate)
- FR13: RustyClaw can rotate sessions with summary injection into the new session
- FR14: RustyClaw can perform daily scheduled rotation at configured time
- FR15: RustyClaw can flush memory on graceful shutdown (SIGTERM)

### Memory Persistence (Declarative)

- FR16: RustyClaw can assemble system prompts from configured markdown files (SOUL.md, USER.md, MEMORY.md, daily files)
- FR17: RustyClaw can index memory files into pgvector embeddings as files change
- FR18: Canti can recall relevant context via semantic similarity search (embeddings) without scanning full files
- FR19: RustyClaw can maintain dual-store consistency between source files and retrieval index
- FR20: RustyClaw can asynchronously index memory file changes into the retrieval store without blocking the active session

### Muscle Memory (Procedural — Habit Dojo)

- FR21: RustyClaw can load Habit Dojo skills into the system prompt based on relevance
- FR22: Canti can query the skill registry for matching habits given a task context
- FR23: Canti can inspect a skill's provenance (origin, reasoning, what it replaced)
- FR24: Canti can install new skills via the Habit Dojo with full provenance metadata
- FR25: RustyClaw can position skills in cache-aware prompt zones for optimal cache hit rates

### Scheduled Operations

- FR26: RustyClaw can execute cron jobs on configured schedules (cron expressions, intervals, one-shot)
- FR27: RustyClaw can persist cron job state across restarts
- FR28: RustyClaw can yield scheduled work to interactive messages (priority queue)
- FR29: Jay can add, modify, or disable cron jobs via configuration

### Observability & Operations

- FR30: Jay can query a health endpoint for system status (uptime, queue depth, context usage, last message time)
- FR31: RustyClaw can emit structured JSON logs for all significant events
- FR32: RustyClaw can hot-reload configuration from file changes without restart
- FR33: RustyClaw can integrate with systemd watchdog (sd_notify)
- FR34: Jay can view operational metrics (cache hit rates, token usage, response latency)

### Prompt Architecture

- FR35: RustyClaw can assemble prompts in zone-based layout (Zone 1: static identity, Zone 2: skills, Zone 3: conversation)
- FR36: RustyClaw can track cache hit rates per zone to inform optimization
- FR37: RustyClaw can collect metrics that feed into future OPT self-tuning

### Risk Mitigation Strategy

**Technical Risks:**

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| `claude -p --resume` changes behavior | Low | Pin CLI version, test on Chaos Monkey. Fallback: session-per-message (degraded but functional) |
| signal-cli daemon unreliable | Medium | Separate watchdog. RustyClaw reconnects on disconnect. Jay gets alerted. |
| Session UUID corruption | Low | Atomic writes. Backup UUID. Worst case: new session (memory files persist on disk) |
| Pre-rotation flush fails | Low | Retry flush up to 3 times. If all fail, abort rotation and alert Jay. Never rotate without confirmed flush. |

**Resource Risks:**

| Risk | Mitigation |
|------|-----------|
| Project stalls | MVP is 12 capabilities, 5-7 weeks. Phase gates prevent scope creep. |
| Canti can't build alone | Jay reviews all PRs. Claude Code sub-agents handle implementation. Vertical TDD keeps slices small. |
| OpenClaw breaks during development | Keep OpenClaw running in parallel. RustyClaw doesn't replace it until MVP passes the 1-week gate. |

**Market Risks:** N/A — single-user product. The only "market" is Jay's satisfaction.

## Non-Functional Requirements

### Performance

- **NFR1:** RustyClaw's processing overhead (message received → Claude invoked, Claude response → message dispatched) must add less than 500ms total, excluding Signal transport and LLM inference time. The two bottlenecks we don't control (Signal delivery, model inference) are explicitly outside this budget.
- **NFR2:** The 500ms processing budget includes all internal operations: queue routing, DB lookups (pgvector embeddings), prompt assembly, file reads, skill matching, and response routing. Localhost PostgreSQL with indexed queries should consume a fraction of this budget.
- **NFR3:** System prompt assembly (file reads + skill loading + zone layout) must complete within 200ms.
- **NFR4:** Embeddings-based memory recall must return results within 500ms.
- **NFR5:** Idle resource usage must remain under 50MB RSS and <1% CPU. Under active load, RustyClaw should leverage available cores for parallelized operations (concurrent indexing, skill matching, message handling). Canti's 125GB RAM and multi-core Ryzen are available — use them when throughput benefits.
- **NFR6:** Async memory indexing must not introduce perceptible latency to the active session.

### Security

- **NFR7:** Private keys and credentials must be stored with permissions no broader than 0600. If insecure permissions are detected at startup, the service starts in degraded mode and injects a system message to Claude requesting corrective action — escalating to Jay if the model cannot resolve it autonomously. The service does not crash on security findings; it declares them. (Declarative intent, LLM is the implementation.)
- **NFR8:** Sender authentication via allowlist must reject messages from unknown senders before any processing occurs, across all channel adapters (Signal and any future providers). Fail-closed: no allowlist configured = no messages processed.
- **NFR9:** No private data (conversation content, credentials, memory files) may be exposed via the health endpoint or structured logs.
- **NFR10:** All outbound messages must pass through a pre-send sanitizer that redacts recognized secret patterns (API keys, tokens, credentials) before dispatch to any channel adapter. Defense in depth — catches accidental exposure from model output before it reaches any communication surface.
- **NFR11:** Log output must redact sensitive values by default using pattern-based sanitization.
- **NFR12:** Configuration files containing secrets must support environment variable references rather than inline values.
- **NFR13:** Claude session content must not be persisted to disk except through explicit memory file writes controlled by the AI.

### Reliability

- **NFR14:** RustyClaw must recover from unexpected restarts without data loss — session UUID and memory files persist on disk.
- **NFR15:** Pre-rotation memory flush must succeed before rotation proceeds — never rotate without confirmed flush (retry up to 3 times, alert Jay on failure).
- **NFR16:** systemd watchdog integration must detect hangs and trigger automatic restart within 30 seconds.
- **NFR17:** Signal adapter must detect disconnection and attempt reconnection with exponential backoff.
- **NFR18:** If Claude API is unreachable, messages must queue and retry — not drop silently. Queue implementation: Tokio bounded channels (POC through MVP), designed behind a trait abstraction (Open/Closed principle) so the backing store can be extended to Redis (via redis-rs crate) or similar when Growth phase introduces fan-out requirements (e.g., MCP servers needing pub/sub for memory write notifications to both embeddings indexer and graph builder).
- **NFR19:** Uptime target: 99.9% (three nines) measured monthly (~8.7 hours/year max downtime). Achieved through: hot-reload for all configuration and prompt files, graceful error handling that never kills the process, skill loading with validity checks before activation, systemd watchdog auto-recovery, and binary updates as the only restart trigger.
- **NFR20:** Configuration hot-reload must validate before applying — invalid config must never crash or degrade the running system. Errors bubble up as warnings, prior valid config remains active.
- **NFR21:** Changes to core files (system prompt, SOUL.md, USER.md, MEMORY.md, skills) must be detected and applied in real-time without restart. Skill mutations must pass validity checks before being loaded into active prompt assembly.
- **NFR22:** On startup, if the configuration file is missing, corrupted, or invalid, RustyClaw must generate a known-good default configuration, start successfully in degraded mode, and inject a system message to Claude: "Running on defaults — configuration needs attention." The service always boots. First-run and disaster recovery are the same code path.

### Integration

- **NFR23:** signal-cli integration must use the documented JSON-RPC protocol — RustyClaw must not manage the signal-cli process lifecycle.
- **NFR24:** Claude CLI integration must use `--output-format json` as the contract — response parsing must handle format changes gracefully.
- **NFR25:** PostgreSQL connections must use connection pooling (sqlx) with configurable pool size.
- **NFR26:** All file writes (config, memory, session state) must be atomic (write-to-temp, rename) to prevent partial reads.
- **NFR27:** systemd integration must use Type=notify with sd_notify for readiness and watchdog signaling. This ties RustyClaw closely to the *nix ecosystem by design.
- **NFR28:** External dependency versions (signal-cli, claude CLI) must be pinnable and tested on Chaos Monkey before production upgrade.
