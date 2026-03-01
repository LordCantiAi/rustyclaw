# 🦀 RustyClaw

A personal AI assistant framework built in Rust. Trait-based, declarative, singleton-session architecture.

**Authors:** [Jay Grissom](https://github.com/jfgrissom) & [Lord Canti](https://github.com/LordCantiAi)

## What Is This?

RustyClaw is a focused AI session orchestrator. It owns the **session lifecycle, communication channels, and operational policies** — delegating intelligence to Claude Code and data services to MCP servers.

It is not a framework. It is not a platform. It is the thin, principled layer between a human and their AI assistant that ensures:

- **Singleton identity** — One assistant, one mind, one continuity
- **Ambient presence** — Lives in your messaging app, not a separate tool
- **Declarative policy** — System prompts, compaction thresholds, and scheduling are configuration, not code
- **Self-tuning operation** — The Ouroboros Prompt Tuner (OPT) optimizes its own cache and prompt strategies

## Design Principles

1. **Functional composition over inheritance** — Traits, not class hierarchies
2. **Declarative intent over imperative code** — YAML config, not hardcoded behavior
3. **Lifecycle events over timer polling** — React to state changes, don't poll for them
4. **Single responsibility** — The orchestrator orchestrates. Search services search. The LLM thinks.
5. **Open/Closed** — Add Signal today, Discord tomorrow. Never modify core.
6. **Singleton identity** — Architecturally enforced. One session, one mind.
7. **Red-Green-Refactor (Vertical TDD)** — One test, one implementation, repeat.

## Architecture

```
RustyClaw Core
├── Session Manager (singleton, lifecycle-aware)
├── Message Queue (ordered, backpressure, priority)
├── Scheduler (cron, timers, lifecycle-aware)
├── Memory Manager (event-driven compaction)
├── Config System (YAML, hot-reload)
└── Observability (structured logs, metrics, health)

Channel Adapters (trait-based)
├── Signal Adapter (signal-cli)
└── [Future: Discord, Telegram, etc.]

Model Adapters (trait-based)
├── Claude Adapter (claude -p --resume)
│   ├── Prompt Optimizer (cache-aware zones)
│   ├── Skill Loader (inline vs rotation)
│   └── OPT Integration (self-tuning)
└── [Future: Ollama, OpenRouter, etc.]

MCP Servers (standalone Rust microservices)
├── memory-search (pgvector)
├── habit-dojo (pgvector)
├── graph-rag (Apache AGE) [future]
└── trading-db [future]
```

## Status

🚧 **Phase 0: Planning & Setup** — Project scaffolding, BMAD artifacts, trait interfaces.

See [docs/](docs/) for full BMAD documentation.

## Building

```bash
cargo build
cargo test
```

## License

TBD

---

*"Structure isn't an impediment to passionate output — it's the road that lets ideas move at high velocity."*
