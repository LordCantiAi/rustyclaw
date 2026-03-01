---
stepsCompleted: [vision, users, metrics, scope]
inputDocuments: [rustyclaw-design-plan.md]
date: 2026-03-01
author: Jay Grissom & Lord Canti
---

# Product Brief: RustyClaw

## Vision

RustyClaw is a focused AI session orchestrator written in Rust that replaces OpenClaw as the operational layer between a human (Jay) and their AI assistant (Canti). It owns session lifecycle, communication channels, and operational policies — delegating intelligence to Claude Code and data services to MCP servers.

### Problem Statement

OpenClaw solves real problems (session orchestration, compaction, scheduling, Signal presence) but carries costs we no longer want to pay:

- **Fork maintenance** — Rebasing custom commits across major versions costs days (v2.6 → v2.23 incident)
- **Crash loops** — 132 loops (Feb 19), 70+ loops (Feb 25) from breaking config changes in upstream
- **Hardcoded system prompt** — Policy embedded in source code, not configuration
- **Opaque compaction** — Memory loss during context compaction with no hooks for pre-flush

### Value Proposition

If we maintain code, it should be **our code** built on **our principles**. RustyClaw is what OpenClaw should have been — for us.

## Target Users

### Primary: Jay Grissom
- Power user, software engineer, daily interaction via Signal
- Needs: ambient AI presence in messaging, reliable scheduling, zero data loss during session transitions
- Context: Uses Canti for trading analysis, coding, research, personal assistance, family coordination

### Secondary: Lord Canti (the AI)
- The assistant instance running within RustyClaw
- Needs: singleton identity enforcement, memory persistence guarantees, skill loading, MCP data access
- Context: Maintains continuity across sessions via file-based memory, serves multiple roles (developer, analyst, family member)

## Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Message round-trip (Signal → response) | ≤ 10s for resumed sessions | Observability metrics |
| Context loss during rotation | Zero | Memory diff before/after rotation |
| Cron job reliability | 100% fire rate | Scheduler logs |
| Cache hit rate | > 80% | Claude response metrics via OPT |
| Uptime | > 99.5% | Health endpoint monitoring |
| Session resume success | 100% | Session Manager logs |
| Memory pre-flush | 100% before every rotation | Memory Manager events |

## Scope

### IN Scope
- Session lifecycle management (create, resume, rotate, flush)
- Signal channel adapter (signal-cli integration)
- Claude model adapter (claude -p --resume)
- Message queue with priority and backpressure
- Cron scheduler (migrate 12 existing jobs)
- Memory manager with event-driven compaction
- Declarative YAML configuration with hot-reload
- Prompt optimizer with cache-aware zone layout
- Ouroboros Prompt Tuner (OPT) — self-tuning feedback loop
- Observability (structured logging, metrics, health endpoint)
- MCP servers: memory-search, habit-dojo

### OUT of Scope (for now)
- Multi-user support (this is Jay + Canti only)
- Web UI or dashboard
- Multiple simultaneous AI models in one session
- GraphRAG MCP server (future phase)
- Trading DB MCP server (future phase)
- Custom domain email hosting
- Public API or SaaS offering

## Technical Constraints

- **Language:** Rust (non-negotiable — our principles, our language)
- **Async runtime:** Tokio
- **Signal:** signal-cli daemon (JSON-RPC or D-Bus)
- **Claude:** CLI invocation via `claude -p --resume` (not direct API — uses Claude Code's tool ecosystem)
- **Database:** PostgreSQL with pgvector (existing infrastructure on Canti/Fozzy)
- **Deployment:** Single binary, systemd service on Canti (192.168.64.86)
- **Config:** YAML files, environment variable interpolation for secrets

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Claude Code `--resume` behavior changes | Low | High | Pin CLI version, test on Chaos Monkey first |
| Signal-cli daemon instability | Medium | Medium | Watchdog restart, health monitoring |
| Session rotation loses context | Medium | High | Pre-rotation flush guarantee, summary validation |
| OPT tunes parameters incorrectly | Low | Medium | Guardrail bounds, max 20% change per run |
| Scope creep | High | Medium | Strict phase gates, MVP-first |

## Origin

Born from a Saturday conversation (Feb 28, 2026) that started with exploring Claude Code Remote as an OpenClaw replacement. Through systematic exploration, we realized we were "slowly rediscovering why OpenClaw was built the way it was" — and decided to build our own, in Rust, on our terms.

Key quote: *"If we maintain code, it should be our code built on our principles."* — Jay
