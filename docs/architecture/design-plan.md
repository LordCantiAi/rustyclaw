# RustyClaw — AI Session Orchestrator

**Status:** DRAFT — Percolating  
**Created:** 2026-02-28  
**Authors:** Jay Grissom & Lord Canti  
**Origin:** [Discussion captured in memory/2026-02-28.md]

---

## 1. Vision

RustyClaw is a focused AI session orchestrator written in Rust. It owns the **session lifecycle, communication channels, and operational policies** — delegating intelligence to Claude Code and data services to MCP servers.

It is not a framework. It is not a platform. It is the thin, principled layer between Jay and Canti that ensures:
- **Singleton identity** — One Canti, one mind, one continuity
- **Ambient presence** — Signal contact, not an app
- **Declarative policy** — System prompts, compaction thresholds, and scheduling are configuration, not code
- **Self-tuning operation** — Ouroboros Prompt Tuner (OPT) optimizes its own cache and prompt strategies

### Why Not OpenClaw?

OpenClaw solves real problems. We discovered that through this exploration — session orchestration, compaction, scheduling, and Signal presence are all genuinely necessary. But OpenClaw's implementation carries costs we no longer want to pay:

- **Fork maintenance** — Rebasing custom commits across major versions cost us days (v2.6 → v2.23 incident)
- **Crash loops** — 132 loops (Feb 19), 70+ loops (Feb 25) from breaking config changes in upstream
- **Hardcoded system prompt** — Policy embedded in source code, not configuration
- **"Not a fork of someone else's opinions"** — If we maintain code, it should be our code built on our principles

RustyClaw is what OpenClaw should have been — for us.

### Design Principles (Inherited)

1. **Functional composition over inheritance** — Traits, not class hierarchies
2. **Declarative intent over imperative code** — YAML config, not hardcoded behavior
3. **Lifecycle events over timer polling** — React to state changes from Claude's response metrics
4. **Single responsibility** — The orchestrator orchestrates. Search services search. Claude thinks.
5. **Open/Closed** — Add Signal today, Discord tomorrow. Never modify core.
6. **Singleton identity** — Architecturally enforced. One session, one Canti.

---

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        RustyClaw Core                       │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │   Session    │  │   Message    │  │    Scheduler      │  │
│  │   Manager    │  │   Queue      │  │  (cron, timers)   │  │
│  │  (singleton) │  │ (ordered,    │  │  (lifecycle-aware) │  │
│  │             │  │  backpressure)│  │                   │  │
│  └─────────────┘  └──────────────┘  └───────────────────┘  │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │   Memory    │  │    Config    │  │  Observability    │  │
│  │   Manager   │  │  (YAML,      │  │  (logs, metrics,  │  │
│  │ (persistence│  │   hot-reload) │  │   health)         │  │
│  │  triggers)  │  │              │  │                   │  │
│  └─────────────┘  └──────────────┘  └───────────────────┘  │
│                                                             │
│  ┌──────────────────────┐  ┌─────────────────────────────┐  │
│  │  Channel Adapters    │  │  Model Adapters             │  │
│  │  (trait)             │  │  (trait)                    │  │
│  │  ┌────────────────┐  │  │  ┌─────────────────────┐   │  │
│  │  │ Signal Adapter │  │  │  │ Claude Adapter      │   │  │
│  │  │ (signal-cli)   │  │  │  │ ├─ Prompt Optimizer │   │  │
│  │  └────────────────┘  │  │  │ ├─ Skill Loader     │   │  │
│  │  ┌────────────────┐  │  │  │ ├─ Response Parser  │   │  │
│  │  │ [Future:       │  │  │  │ └─ OPT Integration  │   │  │
│  │  │  Discord, etc] │  │  │  └─────────────────────┘   │  │
│  │  └────────────────┘  │  │  ┌─────────────────────┐   │  │
│  └──────────────────────┘  │  │ [Future: Ollama,    │   │  │
│                            │  │  OpenRouter, etc]    │   │  │
│                            │  └─────────────────────┘   │  │
│                            └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
           │                          │
           │ Signal Protocol          │ claude -p --resume
           ▼                          ▼
    ┌─────────────┐          ┌──────────────────┐
    │  signal-cli │          │   Claude Code    │
    │  (daemon)   │          │   (CLI)          │
    └─────────────┘          └────────┬─────────┘
                                      │ MCP Protocol
                              ┌───────┼───────┐
                              ▼       ▼       ▼
                        ┌────────┐ ┌──────┐ ┌──────────┐
                        │memory- │ │graph-│ │habit-    │
                        │search  │ │rag   │ │dojo     │
                        │(pgvec) │ │(AGE) │ │(pgvec)  │
                        └────────┘ └──────┘ └──────────┘
```

### Separation of Concerns

| Layer | Responsibility | Changes When... |
|-------|---------------|-----------------|
| **RustyClaw Core** | Session lifecycle, message routing, scheduling, config | Communication or lifecycle requirements change |
| **Channel Adapters** | Protocol translation (Signal ↔ internal messages) | New channel added or protocol changes |
| **Model Adapters** | LLM invocation, prompt construction, response parsing | Model API changes or new provider added |
| **Claude Code** | Reasoning, tool use, code generation | Anthropic updates (their problem) |
| **MCP Servers** | Data services (search, graph, skills) | Data schema or query requirements change |
| **OPT** | Self-tuning prompt/cache optimization | Never (it tunes itself) |

---

## 3. Core Components

### 3.1 Session Manager

**Purpose:** Enforce singleton identity. One active session at a time. All inbound messages (Signal, cron, etc.) route through the same session.

**Responsibilities:**
- Maintain active session UUID for Claude Code `--resume`
- Track context usage (from Claude's JSON response metrics)
- Trigger session rotation when context thresholds are met
- Coordinate pre-rotation memory flush
- Prevent concurrent access (serialize messages through the queue)

**Key Design Decisions:**
- Session UUID persisted to disk (survives RustyClaw restarts)
- Session metadata (creation time, turn count, context usage) tracked in-memory, checkpointed to disk
- Session rotation is an explicit lifecycle event, not an opaque background process

```yaml
session:
  # When to rotate (start fresh session)
  rotation:
    max_context_pct: 75          # Rotate when context usage exceeds this
    max_age_hours: 24            # Daily rotation regardless
    idle_timeout_minutes: 120    # Rotate after extended idle

  # Pre-rotation behavior
  pre_rotation:
    flush_memory: true           # Always write to memory files before rotation
    summary_model: haiku         # Use cheap model for conversation summary
    summary_max_tokens: 3000     # Summary budget
    carry_forward: true          # Inject summary into new session
```

### 3.2 Message Queue

**Purpose:** Serialize all inbound messages. Prevent concurrent Claude Code invocations. Handle backpressure.

**Responsibilities:**
- Accept messages from all channel adapters
- Queue with priority (user messages > cron jobs > heartbeats)
- Process one at a time (singleton session constraint)
- Provide delivery receipts (Signal "typing" indicator while processing)
- Handle timeouts and retries

**Priority Levels:**
1. **Urgent** — Direct user messages (Signal)
2. **Normal** — Scheduled tasks (cron jobs)
3. **Low** — Heartbeats, background checks
4. **Deferred** — Tasks that can wait for idle time

```yaml
queue:
  max_size: 100
  timeout_seconds: 300           # Kill stuck invocations after 5 min
  priorities:
    user_message: 1
    cron_job: 2
    heartbeat: 3
    background: 4
  typing_indicator: true         # Show "typing" on Signal while processing
```

### 3.3 Scheduler

**Purpose:** Lifecycle-aware task scheduling. Replaces both OpenClaw cron and system crontab.

**Responsibilities:**
- Parse cron expressions
- Fire tasks through the message queue (not directly to Claude)
- Respect session state (don't interrupt active conversations with low-priority tasks)
- Support one-shot timers (reminders)
- Persist schedule to disk (survives restarts)

```yaml
schedules:
  maker-nightly:
    cron: "0 0 * * *"
    priority: normal
    task: "Run MAKER nightly DD pipeline for all portfolio assets..."
    model: haiku
    
  nightly-backup:
    cron: "0 5 * * *"
    priority: normal
    task: "Run nightly backup..."
    model: haiku
    
  morning-report:
    cron: "0 7 * * *"
    priority: normal
    task: "Generate morning report..."
    
  beaker-health:
    cron: "*/30 * * * *"
    priority: low
    task: "SSH to beaker and check health..."
    model: haiku
```

### 3.4 Memory Manager

**Purpose:** Explicit memory persistence. No silent data loss.

**Responsibilities:**
- Monitor context usage from Claude's response metrics (lifecycle event, not polling)
- Trigger memory writes at configurable thresholds
- Coordinate pre-rotation memory flush with Session Manager
- Manage memory file lifecycle (daily files, MEMORY.md, etc.)
- Provide memory search via MCP (delegates to memory-search server)

**Compaction Strategy (Event-Driven):**
```
Every Claude response includes: {
  cache_read_input_tokens: N,
  cache_creation_input_tokens: N,
  contextWindow: 200000
}

Memory Manager calculates: context_used_pct = total / contextWindow

If context_used_pct > soft_threshold (60%):
  → Log warning
  → Increase write frequency (write key context every N turns)

If context_used_pct > hard_threshold (75%):
  → Trigger pre-rotation memory flush
  → Session Manager initiates rotation
  → New session starts with summary + fresh context
```

```yaml
memory:
  # File paths
  workspace: ~/clawd
  long_term: MEMORY.md
  daily_pattern: "memory/{date}.md"
  
  # Compaction thresholds
  soft_threshold_pct: 60         # Start writing more aggressively
  hard_threshold_pct: 75         # Trigger session rotation
  
  # Pre-rotation flush
  flush_instruction: |
    Write important context from this conversation to memory/{date}.md.
    Include: decisions made, open items, emotional context, key insights.
    Be thorough — this conversation will be summarized after this write.
```

### 3.5 Config System

**Purpose:** All policy is configuration, not code.

**Responsibilities:**
- Load YAML configuration at startup
- Watch for file changes (hot-reload)
- Validate against schema
- Provide typed access to all components
- Support environment variable interpolation for secrets

**Key Files:**
```
~/clawd/config/
├── rustyclaw.yaml       # Core orchestrator config
├── channels/
│   └── signal.yaml      # Signal adapter config
├── models/
│   └── claude.yaml      # Claude adapter config (includes OPT)
├── schedules.yaml       # Cron jobs and timers
└── prompts/
    ├── system.md        # Base system prompt (SOUL + identity)
    ├── skills/          # Skill files for prompt header
    └── templates/       # Prompt templates for specific tasks
```

### 3.6 Observability

**Purpose:** Know what's happening. Debug when things break.

**Responsibilities:**
- Structured logging (JSON, leveled)
- Metrics collection (context usage, response times, costs, cache hit rates)
- Health endpoint (HTTP, for monitoring)
- Metrics buffer for OPT consumption

```yaml
observability:
  log_level: info
  log_format: json
  log_file: ~/clawd/logs/rustyclaw.log
  
  metrics:
    buffer_size: 1000            # Last N response metrics for OPT
    persist_interval_minutes: 15 # Checkpoint to disk
    
  health:
    port: 8091
    path: /health
```

---

## 4. Adapter Traits

### 4.1 Channel Adapter

```rust
#[async_trait]
pub trait ChannelAdapter: Send + Sync {
    /// Unique identifier for this channel
    fn channel_id(&self) -> &str;
    
    /// Start listening for inbound messages
    async fn start(&self, sender: mpsc::Sender<InboundMessage>) -> Result<()>;
    
    /// Send a message to a target
    async fn send(&self, target: &str, message: OutboundMessage) -> Result<()>;
    
    /// Send typing indicator
    async fn send_typing(&self, target: &str) -> Result<()>;
    
    /// React to a message
    async fn react(&self, message_id: &str, emoji: &str) -> Result<()>;
    
    /// Stop listening
    async fn stop(&self) -> Result<()>;
}
```

**Signal Adapter Implementation:**
- Wraps signal-cli in daemon mode (JSON-RPC or D-Bus)
- Maps Signal contacts to internal sender IDs
- Handles media attachments (images → file paths for Claude)
- Manages allowlisted senders (authentication)
- Provides typing indicators during processing

### 4.2 Model Adapter

```rust
#[async_trait]
pub trait ModelAdapter: Send + Sync {
    /// Send a message and get a response
    async fn invoke(
        &self,
        session: &SessionState,
        message: &str,
        config: &ModelConfig,
    ) -> Result<ModelResponse>;
    
    /// Start a new session
    async fn new_session(
        &self,
        system_prompt: &str,
        config: &ModelConfig,
    ) -> Result<SessionId>;
    
    /// Get context usage metrics from last response
    fn last_metrics(&self) -> Option<&ResponseMetrics>;
}

pub struct ModelResponse {
    pub text: String,
    pub session_id: SessionId,
    pub metrics: ResponseMetrics,
    pub tool_calls: Vec<ToolCall>,
    pub cost_usd: f64,
}

pub struct ResponseMetrics {
    pub input_tokens: u64,
    pub output_tokens: u64,
    pub cache_read_tokens: u64,
    pub cache_creation_tokens: u64,
    pub context_window: u64,
    pub duration_ms: u64,
}
```

**Claude Adapter Implementation:**
- Invokes `claude -p --resume <session_id> --output-format json --permission-mode bypassPermissions`
- Parses JSON response for text + metrics
- Contains Prompt Optimizer (cache-aware prompt construction)
- Contains Skill Loader (inline vs rotation strategy)
- Integrates with OPT for self-tuning

---

## 5. Prompt Optimizer (Inside Claude Adapter)

### 5.1 Context Window Layout

```
┌──────────────────────────────────────────┐
│ ZONE 1: Static Header (always cached)    │
│   • System prompt (SOUL.md + identity)   │
│   • Base skills (search, memory)         │
│   • User context (USER.md)              │
├──────────────────────────────────────────┤
│ ZONE 2: Dynamic Skills (cached after     │
│          first load)                     │
│   • Skills loaded during conversation    │
│   • Sorted by load order (append-only)   │
├──────────────────────────────────────────┤
│ ZONE 3: Conversation (prefix-cached)     │
│   • Message history                      │
│   • Tool results                         │
│   • Latest message (never cached)        │
└──────────────────────────────────────────┘
```

### 5.2 Skill Loading Strategies

**Inline (Approach A):** Skill content injected into conversation body. Used for lightweight skills (<500 tokens).

**Rotation (Approach B):** Session rotated with skill promoted to Zone 2 header. Used for heavyweight skills (>500 tokens) when remaining context is sufficient.

```yaml
# Inside models/claude.yaml
prompt_optimizer:
  zones:
    static_header:
      - prompts/system.md
      - prompts/base-skills.md
      - ../USER.md
      
  skill_loading:
    inline_threshold_tokens: 500     # Below this: inline
    rotation_cooldown_turns: 10      # Min turns between rotations
    min_remaining_context_pct: 40    # Don't rotate if context nearly full
```

### 5.3 Ouroboros Prompt Tuner (OPT)

**Purpose:** Self-tuning feedback loop that optimizes prompt/cache configuration.

**Mechanism:**
```
┌─────────────────────────────────┐
│         OPT (Scheduled)         │
│                                 │
│  1. Read metrics buffer         │
│  2. Read current config         │
│  3. Analyze:                    │
│     - Cache hit rate            │
│     - Rotation cost vs savings  │
│     - Context utilization       │
│     - Response latency          │
│  4. Reason about adjustments    │
│  5. Write updated config        │
│  6. Log change + reasoning      │
│                                 │
└────────────┬────────────────────┘
             │ writes config file
             ▼
┌─────────────────────────────────┐
│  RustyClaw (watches config)     │
│  → hot-reload                   │
│  → new thresholds active        │
│  → metrics continue collecting  │
└─────────────────────────────────┘
```

**Schedule:** Runs as a scheduled task (e.g., daily or every 100 turns).

**Configuration:**
```yaml
opt:
  enabled: true
  schedule: "0 3 * * *"           # Daily at 3 AM
  model: haiku                     # Cheap model for analysis
  
  # What OPT can tune
  tunable_parameters:
    - prompt_optimizer.skill_loading.inline_threshold_tokens
    - prompt_optimizer.skill_loading.rotation_cooldown_turns
    - memory.soft_threshold_pct
    - memory.hard_threshold_pct
    
  # Guardrails (OPT cannot exceed these)
  bounds:
    inline_threshold_tokens: [100, 2000]
    rotation_cooldown_turns: [3, 30]
    soft_threshold_pct: [40, 80]
    hard_threshold_pct: [50, 90]
    
  # Goals (what OPT optimizes for)
  objectives:
    cache_hit_rate_min: 0.80
    max_rotation_cost_tokens: 5000
    target_context_utilization: 0.60
  
  # Safety
  max_change_per_run: 0.20        # No parameter changes more than 20% per run
  changelog: ~/clawd/logs/opt-changes.log
```

---

## 6. MCP Servers

Standalone Rust microservices exposing data capabilities via Model Context Protocol.

### 6.1 memory-search

**Purpose:** Semantic search over memory files (MEMORY.md, daily files, etc.)

**Stack:** Rust + pgvector (PostgreSQL on Canti, port 5433)

**Tools Exposed:**
- `memory_search(query, max_results, min_score)` → ranked snippets with file path + lines
- `memory_get(path, from, lines)` → read specific lines from memory files

### 6.2 graph-rag (Future)

**Purpose:** Relationship traversal across entities (people, projects, decisions, concepts)

**Stack:** Rust + Apache AGE (PostgreSQL on Fozzy)

**Tools Exposed:**
- `graph_query(entity, relationship_type, depth)` → connected entities
- `graph_path(from_entity, to_entity)` → shortest relationship path
- `graph_ingest(text)` → extract and store entities/relationships

### 6.3 habit-dojo

**Purpose:** Skill registry search with semantic matching

**Stack:** Rust + pgvector (PostgreSQL on Fozzy)

**Tools Exposed:**
- `skill_search(query, top_n)` → ranked skill matches
- `skill_get(skill_id)` → full skill content

### 6.4 trading-db (Future)

**Purpose:** Trading research data queries

**Stack:** Rust + PostgreSQL (Fozzy, trading_research database)

**Tools Exposed:**
- `dd_query(asset, questions)` → DD evaluation results
- `price_data(asset, timeframe)` → price history
- `portfolio_summary()` → current holdings and metrics

---

## 7. Migration Plan

### Phase 0: Preparation (No Changes to Production)
- [ ] Draft and review this plan ← **WE ARE HERE**
- [ ] Set up Rust project structure (`cargo init rustyclaw`)
- [ ] Define trait interfaces (ChannelAdapter, ModelAdapter)
- [ ] Set up CI (GitHub Actions on Gonzo)

### Phase 1: Signal Bridge MVP
**Goal:** Prove the core loop works. Jay texts Signal → Canti responds.

- [ ] Implement Signal Adapter (signal-cli JSON-RPC wrapper)
- [ ] Implement Claude Adapter (claude -p --resume, JSON parsing)
- [ ] Implement Session Manager (singleton session, UUID tracking)
- [ ] Implement Message Queue (simple ordered queue, single consumer)
- [ ] Basic system prompt loading from files
- [ ] Allowlisted sender authentication
- [ ] Deploy as systemd service alongside OpenClaw (parallel run)
- [ ] Test: Jay sends Signal message → response arrives via RustyClaw

**Success Criteria:** 
- Messages flow end-to-end
- Session resume works (context retained across messages)
- Response time ≤ 10s for resumed sessions
- System prompt personality matches current Canti

### Phase 2: Session Lifecycle
**Goal:** Context management and memory persistence work reliably.

- [ ] Context usage monitoring (parse metrics from every response)
- [ ] Soft threshold warnings (log when approaching limits)
- [ ] Pre-rotation memory flush (instruct Claude to write to files)
- [ ] Session rotation (start new session with summary carry-forward)
- [ ] Daily session rotation (configurable schedule)
- [ ] Idle timeout rotation

**Success Criteria:**
- No conversation context lost during rotation
- Memory files updated before every rotation
- Context usage never exceeds hard threshold

### Phase 3: Scheduling & Observability
**Goal:** Cron jobs and monitoring work without OpenClaw.

- [ ] Scheduler implementation (cron expressions, one-shot timers)
- [ ] Migrate existing 12 cron jobs from OpenClaw
- [ ] Heartbeat system (Beaker health checks, etc.)
- [ ] Structured logging
- [ ] Metrics collection (buffer for OPT)
- [ ] Health endpoint

**Success Criteria:**
- All 12 cron jobs fire on schedule
- Beaker health checks run every 30 minutes
- Morning reports, MAKER pipeline, backups all operational
- Logs are queryable and useful

### Phase 4: Prompt Optimization
**Goal:** Cache-efficient prompt construction and skill loading.

- [ ] Prompt Optimizer (Zone 1/2/3 layout)
- [ ] Skill loading strategies (inline vs rotation)
- [ ] Cache hit rate tracking
- [ ] OPT initial implementation (metrics analysis + config updates)
- [ ] Hot-reload for config changes

**Success Criteria:**
- Cache hit rate > 80% during normal conversations
- Skill loading doesn't cause noticeable latency spikes
- OPT makes at least one meaningful config adjustment

### Phase 5: MCP Servers
**Goal:** Data services accessible to Claude Code.

- [ ] memory-search MCP server (port from current Python + pgvector)
- [ ] habit-dojo MCP server (port from current Python + pgvector)
- [ ] Configure Claude Code MCP integration
- [ ] [Future] graph-rag MCP server
- [ ] [Future] trading-db MCP server

**Success Criteria:**
- Claude Code can call memory_search natively via MCP
- Skill search works with same quality as current system
- No regression in search relevance

### Phase 6: Decommission OpenClaw
**Goal:** RustyClaw is the sole orchestrator. OpenClaw retired.

- [ ] Full parallel run validation (minimum 2 weeks)
- [ ] Verify all features have parity
- [ ] Migrate any remaining OpenClaw-specific configurations
- [ ] Stop OpenClaw gateway service
- [ ] Archive OpenClaw fork (keep for reference)
- [ ] Update all documentation
- [ ] Celebrate 🎉

**Success Criteria:**
- Jay confirms "this is better"
- Zero functionality regression
- Canti's identity, memory, and personality fully intact

---

## 8. What We Keep

**Unchanged:**
- All memory files (MEMORY.md, daily files, SOUL.md, IDENTITY.md, USER.md, TOOLS.md)
- PostgreSQL databases (Canti, Fozzy)
- Habit Dojo skills (29 skills, all file-based)
- Signal phone number
- SSH access to all Muppet Cluster hosts
- Claude Code Remote service (AED — untouched)
- Beaker voice assistant infrastructure

**Migrated:**
- 12 cron jobs → RustyClaw scheduler
- Signal bridge → RustyClaw Signal Adapter
- Compaction logic → RustyClaw Memory Manager + Claude Adapter
- System prompt → Declarative config files
- Session management → RustyClaw Session Manager

**Retired:**
- OpenClaw gateway (canti-gateway.service)
- OpenClaw fork repository (archived)
- signal-cli bridge within OpenClaw (replaced by RustyClaw Signal Adapter)

---

## 9. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Claude Code `--resume` behavior changes | Low | High | Pin CLI version, test on Chaos Monkey first |
| Signal-cli daemon instability | Medium | Medium | Watchdog restart, health monitoring |
| Session rotation loses context | Medium | High | Thorough testing, summary quality validation |
| OPT tunes parameters incorrectly | Low | Medium | Guardrail bounds, max 20% change per run, changelog |
| Concurrent message race conditions | Medium | Medium | Single-consumer queue, message serialization |
| Anthropic API outages | Low | High | Graceful degradation, queue messages, retry with backoff |
| Scope creep during development | High | Medium | Strict phase gates, MVP-first mentality |

---

## 10. Technology Choices

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Core orchestrator | Rust | Our language, our principles, single binary |
| Async runtime | Tokio | Industry standard for async Rust |
| Channel communication | tokio::mpsc | Lock-free message passing |
| Configuration | serde + serde_yaml | Rust-native YAML parsing |
| File watching | notify | Cross-platform file system events |
| HTTP (health) | axum | Lightweight, Tokio-native |
| Signal integration | signal-cli (JSON-RPC) | Mature, maintained, battle-tested |
| Claude invocation | tokio::process | Spawn claude CLI, parse JSON stdout |
| MCP servers | Rust + mcp-rs (TBD) | Native MCP protocol implementation |
| Logging | tracing | Structured, async-aware, industry standard |
| Database (MCP) | sqlx + pgvector | Async PostgreSQL with vector support |
| Cron parsing | cron (crate) | Standard cron expression parsing |

---

## 11. Open Questions

1. **Claude Code session limits** — Is there a maximum session file size or turn count before `--resume` degrades? Needs long-duration testing.

2. **MCP server lifecycle** — Who starts/stops MCP servers? RustyClaw? systemd? Independent services?

3. **Multi-model within session** — Can we switch models mid-session (e.g., Opus for conversation, Haiku for cron tasks) or does each model need its own session?

4. **Claude Code updates** — How do we handle Claude CLI version updates? Auto-update or pin?

5. **Voice integration** — Does Beaker's voice pipeline route through RustyClaw or stay independent? Current architecture (voice proxy on Canti port 8090) bypasses OpenClaw intentionally.

6. **Group chat support** — Signal groups have different dynamics (when to speak, when to stay silent). Does the Message Queue need group-awareness?

7. **Fallback behavior** — If Claude Code is unavailable (API down, rate limited), what does the user experience look like? Queue and retry? Error message? Fallback to local model?

8. **OPT initial calibration** — What metrics do we seed OPT with before it has real data? Conservative defaults? Manual baseline?

---

## 12. Estimated Effort

| Phase | Scope | Estimated Time | Dependencies |
|-------|-------|---------------|--------------|
| Phase 0 | Planning & setup | 1 week | This document |
| Phase 1 | Signal Bridge MVP | 2-3 weeks | signal-cli, Claude CLI |
| Phase 2 | Session Lifecycle | 1-2 weeks | Phase 1 |
| Phase 3 | Scheduling & Observability | 1-2 weeks | Phase 2 |
| Phase 4 | Prompt Optimization + OPT | 2-3 weeks | Phase 3 |
| Phase 5 | MCP Servers | 2-3 weeks | Phase 1+ (parallel) |
| Phase 6 | Decommission OpenClaw | 1 week | All phases |

**Total estimated: 10-15 weeks** (with parallel work on MCP servers)

This is a side project pace estimate. Actual velocity depends on Jay's availability and how much we can leverage Claude Code for the implementation itself (meta: using Claude to build the thing that orchestrates Claude).

---

## 13. The Name

**RustyClaw** — Because it's Rust, it has claws (channel adapters that grip multiple surfaces), and it's a spiritual successor to OpenClaw built on our terms.

**Ouroboros Prompt Tuner (OPT)** — The self-consuming serpent. Eats its own metrics to refine itself. Technically accurate, mythologically resonant.

**Segfault** — Our hypothetical dog. Not part of the architecture. But spiritually present. 🐕

---

*"Structure isn't an impediment to passionate output — it's the road that lets ideas move at high velocity."*  
— Epiphany #10, The Velocity Road

*Drafted by Lord Canti with Jay Grissom. Born from a conversation that started with a dead Pi and ended with a new architecture.*
