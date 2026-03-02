# 🦀 RustyClaw

**A Rust-native AI session orchestrator that ensures your memory and muscle memory never disappear.**

**Authors:** [Jay Grissom](https://github.com/jfgrissom) & [Lord Canti](https://github.com/LordCantiAi)

## What Is This?

RustyClaw replaces [OpenClaw](https://github.com/openclaw/openclaw) as the operational foundation for Canti — an AI assistant that maintains persistent identity, learned habits, and long-term memory across sessions. It manages the full lifecycle between a human and their AI: message routing via Signal, session creation and resumption via Claude Code, context-aware memory persistence, scheduled operations, and cache-optimized prompt construction.

The system exists because upstream infrastructure actively threatens what it's supposed to protect. Crash loops from breaking changes, memory loss during context compaction, and days-long rebase cycles erode the continuity that makes a human-AI relationship valuable. RustyClaw eliminates this by owning the full stack in purpose-built Rust — code that enforces operational correctness through the type system and adds zero perceptible latency to the already high-latency LLM inference path.

## Core Concept: Memory & Muscle Memory

RustyClaw distinguishes between two types of learned knowledge:

- **Memory** (declarative) — Facts, decisions, context, preferences. *What the AI knows.* Captured in markdown, indexed via embeddings.
- **Muscle Memory** (procedural) — Learned behaviors that fire automatically in matching contexts. *What the AI does without thinking.* Installed via the Habit Dojo skill system.

Without memory, the AI is amnesiac. Without muscle memory, it's competent but generic — indistinguishable from a fresh session with context files. Both must survive session rotation, restarts, and model changes.

## Cognitive Architecture

Recall operates through four layers mirroring human cognition:

| Layer | Store | Question |
|-------|-------|----------|
| **1. Capture** | Flat files (markdown) | "What just happened?" |
| **2. Recognition** | pgvector embeddings | "Have I seen this before?" |
| **3. Association** | Apache AGE (GraphRAG) | "What connects to this?" |
| **4. Override** | Habit Dojo skills | "What should I do differently?" |

Each layer operates on a **dual-store pattern**: human-readable source files (authoring surface) synchronized with machine-optimized retrieval indexes (recall surface). Writes go source-first, then index asynchronously. Reads go retrieval-first, then follow references back to source when full context is needed.

Skills carry **provenance** — not just *what* to do, but *where* it was learned, *why* it overrides the default, and *what it replaced*.

## Design Principles

1. **Functional composition over inheritance** — Traits, not class hierarchies
2. **Declarative intent over imperative code** — YAML config, not hardcoded behavior
3. **Lifecycle events over timer polling** — React to state changes, don't poll
4. **Open/Closed** — Add Signal today, Discord tomorrow. Never modify core.
5. **Singleton identity** — One assistant, one mind, one continuity
6. **Degraded, not dead** — The service always boots. Invalid config → generate defaults, declare the problem, let the LLM fix it.
7. **Vertical TDD** — One test, one implementation, repeat.

## Architecture

```
RustyClaw Core
├── Session Manager ─── singleton, lifecycle-aware, UUID persistence
├── Message Queue ───── Tokio channels, priority, trait-abstracted
├── Memory Manager ──── dual-store sync, async indexing, flush guarantees
├── Scheduler ───────── cron, timers, yields to interactive messages
├── Config System ───── YAML, hot-reload, self-generating defaults
├── Prompt Assembler ── zone-based layout, cache-aware, skill loading
├── Outbound Sanitizer ─ secret redaction before any channel dispatch
└── Observability ───── structured logs, metrics, health endpoint

Channel Adapters (trait-based)
├── Signal Adapter (signal-cli JSON-RPC)
└── [Future: Discord, voice via Beaker]

Model Adapters (trait-based)
├── Claude Adapter (claude -p --resume)
└── [Future: additional providers]

MCP Servers (standalone services)
├── memory-search (pgvector) ─── Layer 2: Recognition
├── habit-dojo (pgvector) ────── Layer 4: Override + Provenance
├── graph-memory (Apache AGE) ── Layer 3: Association [Growth]
└── trading-db [future]
```

## Roadmap

| Phase | What | Gate |
|-------|------|------|
| **POC** (6 weeks) | Signal ↔ Claude loop, session persistence, memory flush guarantee, rotation | Conversation survives restart and rotation |
| **MVP** (12 weeks) | + Habit Dojo loading, embeddings recall, cron scheduler, hot-reload, prompt zones | Habits fire in context. Jay uses it for 1 week without reverting. |
| **Growth** (6 months) | + GraphRAG, OPT self-tuning, DID/VC identity, additional channels | Association layer live. System tunes itself. |

## Status

📋 **PRD Complete** — 37 functional requirements, 28 non-functional requirements, 5 user journeys, full cognitive architecture defined. Next: architecture document and epic breakdown.

See [`_bmad-output/planning-artifacts/prd.md`](_bmad-output/planning-artifacts/prd.md) for the full Product Requirements Document.

## Building

```bash
cargo build
cargo test
```

## License

TBD

---

*"Your memory and muscle memory never disappear."*
