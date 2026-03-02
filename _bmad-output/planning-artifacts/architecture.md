---
stepsCompleted: ['step-01-init', 'step-02-context']
inputDocuments: ['_bmad-output/planning-artifacts/prd.md', '_bmad-output/planning-artifacts/implementation-readiness-report-2026-03-02.md', 'docs/bmad/product-brief.md', 'docs/architecture/design-plan.md', 'docs/research/did-provenance-research.md']
workflowType: 'architecture'
project_name: 'RustyClaw'
user_name: 'Jay and Canti'
date: '2026-03-02'
---

# Architecture Decision Document

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

## Architecture Decision #1: Direct Anthropic API + Docker MCP Gateway

**Date:** 2026-03-02
**Status:** Accepted
**Participants:** Jay Grissom, Canti

### Context

RustyClaw needs to invoke Claude for reasoning and tool execution. Three options were evaluated:

1. **Claude Code CLI** (`claude -p --resume <uuid>`) — subprocess per message
2. **Direct Anthropic API** (HTTP, persistent connection) — full control
3. **OpenClaw-style gateway** — proxy layer (what we run today)

### Analysis

**Claude Code CLI findings:**
- Every message is a cold-start subprocess (spawn → load session → respond → exit)
- Session state stored in Claude CLI's local storage (~/.claude/) — we don't own it
- ~3-5s process startup overhead per message
- Provides built-in tools (file I/O, exec, web search, memory) for free
- Cost covered by Max plan ($100/mo fixed)
- Session is a cache, not a store — if CLI updates or files corrupt, conversation history lost

**Direct Anthropic API findings:**
- Persistent HTTP connection, no cold start
- Server-side context compaction now available (`compact-2026-12` beta) — 58.6% token reduction, up to 10M token conversations, custom preservation instructions
- Prompt caching: cached tokens = 10% of input cost (our 89% cache hit rate makes this massive)
- Full model routing control: Opus for interactive, Haiku for heartbeats/cron
- We own conversation history (PostgreSQL) — no dependency on CLI storage
- No built-in tools — must provide all tool implementations

**Docker MCP Toolkit findings:**
- 300+ verified MCP servers in Docker catalog (filesystem, shell, brave-search, git, postgres, etc.)
- MCP Gateway: single connection point that manages all servers
- Dynamic MCP: agent can discover and add tools mid-conversation
- Code Mode (experimental): agent writes JavaScript tools, Docker sandboxes execution
- Containerized isolation, credential management, lifecycle handling
- Works with any MCP client — including direct API integration
- Available on Docker Engine (Linux) without Docker Desktop via CLI plugin

**Cost analysis (based on actual Feb 1 - Mar 2 usage):**

| Category | Monthly Calls | Est. API Cost |
|----------|--------------|---------------|
| Opus (interactive) | ~150 | ~$26 |
| Haiku (heartbeats + cron) | ~1,740 | ~$26 |
| **Total** | **~1,890** | **~$52/mo** |

Even at 50% overestimate: ~$78/mo. Within $80 API cap.

**Cost structure:** $20/mo Pro (Jay's personal Claude) + $80/mo API cap = $100/mo total. Saves $100/mo vs current $200 Max plan. Jay retains personal Claude access.

### Decision

**Use the Anthropic Messages API directly. Provide tools via Docker MCP Gateway.**

```
Signal ←→ RustyClaw (orchestrator) ←→ Anthropic API (direct, persistent)
               ↕
         Docker MCP Gateway (one connection)
               ↕
    ┌──────────┼──────────────┐
    ↓          ↓              ↓
mcp/filesystem mcp/brave     Custom MCPs
mcp/shell      mcp/git       (memory, habits, trading)
mcp/postgres
```

### Rationale

1. **No cold start** — persistent HTTP connection vs subprocess per message
2. **Model routing** — Opus for Jay, Haiku for cron, per-message control
3. **Cost control** — pay per token with caching; $52-78/mo estimated vs $100/mo fixed
4. **Custom compaction** — API's `compact-2026-12` lets us instruct "preserve identity/memory content" during compaction
5. **Session ownership** — conversation history in our PostgreSQL, not Claude CLI's local storage
6. **Tool injection control** — Docker MCP Gateway = one tool endpoint that delegates to many. Reduces context window bloat vs injecting all tool schemas.
7. **No CLI version coupling** — API contract is stable, documented, versioned
8. **Docker MCP fills the tool gap** — filesystem, shell, web search, git, postgres all available as verified containerized servers. No wheel reinvention.

### Trade-offs Accepted

- **We build the tool dispatch loop** — receive `tool_use` from API → route to MCP server → return `tool_result`. This is core RustyClaw work.
- **We manage conversation history** — already planned (FR8-FR16, PostgreSQL-backed sessions)
- **We implement compaction triggering** — API provides server-side compaction, we just set the threshold parameter
- **Docker dependency** — Docker Engine required on host. Acceptable given we already run Docker (NetBox on Kermit, potential Kermit containers).

### Implications for PRD

- FR8-FR16 (Session Lifecycle): Session history owned by RustyClaw in PostgreSQL, not delegated to CLI
- FR37 (Prompt Assembly): Zone-based layout maps directly to API's system/user message structure with cache_control headers
- FR38 (Cache Tracking): API returns cache hit metrics per response — direct observability
- New: ModelAdapter trait has `AnthropicApiAdapter` (primary), no `ClaudeCliAdapter` needed
- New: MCP client integration required — Rust MCP client crate for Docker Gateway communication

### Migration Path

Docker MCP Toolkit can be deployed on OpenClaw today (before RustyClaw exists) to validate the tool ecosystem and start getting immediate benefit.

## Project Context Analysis

### Requirements Overview

**Functional Requirements (39 FRs, 7 capability areas):**

- **Message Routing (FR1-FR7):** Signal send/receive/typing/reactions, priority queue, sender auth. Architectural implication: channel adapter trait + message queue + auth middleware.
- **Session Lifecycle (FR8-FR16):** Create/resume/persist/monitor/flush/rotate sessions, graceful SIGTERM. Architectural implication: session state machine (enum-driven), flush-before-rotate enforcement, UUID persistence. History owned in PostgreSQL.
- **Memory Persistence (FR17-FR21):** Prompt assembly from files, pgvector indexing, semantic recall, dual-store consistency, async indexing. Architectural implication: memory manager with file watcher + embedding pipeline + dual-store sync abstraction.
- **Muscle Memory (FR22-FR26):** Habit Dojo skill loading, registry queries, provenance inspection, skill installation, cache-aware positioning. Architectural implication: skill loader integrated with prompt assembler, MCP server interface for registry.
- **Scheduling (FR27-FR30):** Cron execution, state persistence, priority yielding, config-driven jobs. Architectural implication: scheduler enqueues to same message queue, persistent job store.
- **Observability (FR31-FR36):** Health endpoint, structured logs, hot-reload, watchdog, default config generation, metrics. Architectural implication: Axum HTTP server, tracing crate, notify file watcher, sd_notify integration.
- **Prompt Architecture (FR37-FR39):** Zone-based layout with cache_control headers, per-zone cache tracking via API response metrics, OPT metrics collection.

**Non-Functional Requirements (28 NFRs, 4 categories):**

- **Performance (6):** < 500ms total overhead, < 200ms prompt assembly, < 500ms recall, < 50MB idle. Drives: async-first architecture, connection pooling, parallel indexing.
- **Security (7):** Permission enforcement, outbound sanitizer, log redaction, env var secrets, fail-closed auth. Drives: sanitizer pipeline, auth middleware, config interpolation.
- **Reliability (9):** 99.9% uptime, flush guarantee, watchdog, reconnection, self-generating defaults, hot-reload with validation. Drives: state machine rigor, file watcher, graceful error propagation.
- **Integration (6):** signal-cli JSON-RPC, Anthropic Messages API (HTTP), sqlx pooling, atomic writes, systemd sd_notify, Docker MCP Gateway (MCP protocol over stdio/SSE).

### Scale & Complexity

- **Primary domain:** System daemon — AI session orchestration
- **Complexity level:** Medium-high
- **Architectural components:** ~12 modules
- **Deployment:** Single binary, single machine, systemd-managed
- **Users:** 1 human (Jay) + 1 AI (Canti)
- **Data stores:** Filesystem (markdown) + PostgreSQL (pgvector, future AGE)
- **External services:** signal-cli (Java daemon), Anthropic API (HTTPS), Docker MCP Gateway (containerized MCP servers)

### Technical Constraints & Dependencies

| Constraint | Impact |
|-----------|--------|
| Anthropic Messages API is stateless | RustyClaw owns conversation history in PostgreSQL. Each API call sends full context (system prompt + history + new message). Prompt caching makes repeated context cheap (10% of input cost). |
| signal-cli is a separate daemon | Channel adapter connects via JSON-RPC. RustyClaw does not start/stop signal-cli. Must handle disconnection gracefully. |
| Single-consumer for Claude | Only one message in-flight at a time per session. Queue enforces single-consumer. Cron yields to interactive. |
| Docker MCP Gateway manages tool lifecycle | Tools run in containers. Gateway handles startup, credential injection, isolation. RustyClaw connects as MCP client. |
| Brownfield context | 5+ weeks of operational knowledge from OpenClaw informs every decision. Existing memory files, skills, cron jobs, and config patterns must be compatible. |
| Rust ecosystem | tokio (async), serde (config), tracing (logging), axum (HTTP), sqlx (PostgreSQL), notify (file watching), reqwest (Anthropic API), rmcp or similar (MCP client). |

### Cross-Cutting Concerns

| Concern | Touches |
|---------|---------|
| **File watching** | Config hot-reload, memory file indexing, skill mutation detection, prompt file changes |
| **Atomic writes** | Config, memory files, session history, cron state, metrics checkpoints |
| **Structured logging** | Every module — consistent JSON format via tracing crate |
| **Trait-based adapters** | Channel (Signal, future Discord), Model (Anthropic API, future others), Queue (Tokio, future Redis), MCP Client (Docker Gateway) |
| **Async everywhere** | All I/O: file reads, DB queries, HTTP requests, signal-cli communication, MCP tool calls |
| **Graceful degradation** | Config invalid → defaults. MCP server down → skip tool. API error → queue and retry. Permissions wrong → declare to Claude. |
| **Outbound sanitization** | All channel adapters dispatch through sanitizer. All log output through redactor. |
| **Enum-driven state** | Session state, flush state, config validity, queue priority — all typed enums, never strings. |
| **Prompt caching optimization** | System prompt + identity files in Zone 1 (static, cached). Skills in Zone 2 (semi-static). Conversation in Zone 3 (dynamic). Cache hit rate is a first-class metric. |
