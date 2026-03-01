---
stepsCompleted: [prerequisites, epics, stories]
inputDocuments: [product-brief.md, design-plan.md]
date: 2026-03-01
author: Lord Canti
---

# RustyClaw — Epic Breakdown

## Overview

This document decomposes the RustyClaw design plan into implementable epics and stories using BMAD methodology. Each epic maps to a migration phase, with stories broken into atomic, testable units following vertical TDD (Red-Green-Refactor).

## Requirements Inventory

### Functional Requirements

- FR1: Receive messages from Signal and route to Claude
- FR2: Send Claude responses back via Signal
- FR3: Maintain singleton session (one active Claude session at all times)
- FR4: Resume sessions across RustyClaw restarts
- FR5: Monitor context usage from Claude response metrics
- FR6: Flush memory to files before session rotation
- FR7: Rotate sessions when context thresholds exceeded
- FR8: Execute cron jobs on schedule through message queue
- FR9: Load system prompt from declarative config files
- FR10: Hot-reload configuration changes without restart
- FR11: Construct cache-efficient prompts (zone layout)
- FR12: Load skills inline or via rotation based on weight
- FR13: Self-tune prompt/cache parameters via OPT
- FR14: Expose health endpoint for monitoring
- FR15: Structured logging for all operations
- FR16: Authenticate senders against allowlist
- FR17: Priority-based message queue with backpressure
- FR18: Typing indicator on Signal during processing

### Non-Functional Requirements

- NFR1: Message round-trip ≤ 10s for resumed sessions
- NFR2: Zero context loss during session rotation
- NFR3: Single binary deployment (no runtime dependencies except signal-cli and claude)
- NFR4: Configuration changes apply without downtime
- NFR5: Survive RustyClaw process restarts (session UUID persisted)
- NFR6: Memory pre-flush 100% guaranteed before every rotation
- NFR7: Graceful degradation when Claude is unavailable

## FR Coverage Map

| FR | Epic |
|----|------|
| FR1, FR2, FR16, FR18 | Epic 1: Signal Bridge MVP |
| FR3, FR4, FR5, FR9 | Epic 1: Signal Bridge MVP |
| FR17 | Epic 1: Signal Bridge MVP |
| FR5, FR6, FR7 | Epic 2: Session Lifecycle |
| FR8 | Epic 3: Scheduling & Observability |
| FR14, FR15 | Epic 3: Scheduling & Observability |
| FR10, FR11, FR12 | Epic 4: Prompt Optimization |
| FR13 | Epic 4: Prompt Optimization |

## Epic List

1. **Signal Bridge MVP** — Prove the core loop: Signal → Queue → Claude → Signal
2. **Session Lifecycle** — Context management, memory persistence, rotation
3. **Scheduling & Observability** — Cron jobs, logging, metrics, health
4. **Prompt Optimization & OPT** — Cache-efficient prompts, self-tuning
5. **MCP Servers** — Data services for Claude (memory-search, habit-dojo)
6. **Decommission OpenClaw** — Parallel validation, migration, retirement

---

## Epic 1: Signal Bridge MVP

**Goal:** Prove the core loop works. Jay texts Signal → Canti responds via RustyClaw.

**Success Criteria:**
- Messages flow end-to-end through RustyClaw
- Session resume works (context retained across messages)
- Response time ≤ 10s for resumed sessions
- System prompt personality matches current Canti

### Story 1.1: Project Scaffolding

As a developer,
I want a properly structured Rust project with CI,
So that we have a foundation to build on.

**Acceptance Criteria:**

**Given** a fresh checkout of the rustyclaw repo
**When** I run `cargo build`
**Then** the project compiles with zero warnings
**And** `cargo test` runs (even if no tests yet)
**And** `cargo clippy` passes clean

**Tasks:**
- `cargo init rustyclaw` with workspace structure
- Add dependencies: tokio, serde, serde_yaml, tracing, axum, async-trait
- Configure CI (GitHub Actions)
- Add .gitignore, LICENSE, rustfmt.toml, clippy.toml

### Story 1.2: Configuration System

As the orchestrator,
I want to load configuration from YAML files,
So that all policy is declarative and not hardcoded.

**Acceptance Criteria:**

**Given** a valid `rustyclaw.yaml` config file
**When** the application starts
**Then** all configuration sections are parsed and validated
**And** missing required fields produce clear error messages
**And** environment variable interpolation works for secrets (`${VAR_NAME}`)

**Given** an invalid config file
**When** the application starts
**Then** it fails with a descriptive error and does not proceed

### Story 1.3: Channel Adapter Trait

As a developer,
I want a trait definition for channel adapters,
So that Signal (and future channels) plug in without modifying core.

**Acceptance Criteria:**

**Given** the ChannelAdapter trait is defined
**When** I implement it for a mock channel
**Then** the mock can send and receive messages through the core message flow
**And** the trait enforces: start, stop, send, send_typing, react

### Story 1.4: Model Adapter Trait

As a developer,
I want a trait definition for model adapters,
So that Claude (and future models) plug in without modifying core.

**Acceptance Criteria:**

**Given** the ModelAdapter trait is defined
**When** I implement it for a mock model
**Then** the mock can receive messages and return responses with metrics
**And** the trait enforces: invoke, new_session, last_metrics

### Story 1.5: Message Queue

As the orchestrator,
I want a priority message queue,
So that messages are processed in order with backpressure.

**Acceptance Criteria:**

**Given** multiple messages arrive simultaneously
**When** they are enqueued with different priorities
**Then** higher priority messages are processed first
**And** the queue enforces single-consumer (no concurrent processing)
**And** stuck messages are killed after configurable timeout

### Story 1.6: Signal Adapter Implementation

As a user (Jay),
I want to send messages via Signal and receive responses,
So that Canti lives in my messaging app.

**Acceptance Criteria:**

**Given** signal-cli is running in daemon mode
**When** Jay sends a message from his allowlisted number
**Then** the message arrives in the RustyClaw message queue
**And** a typing indicator is sent while processing

**Given** a response is ready from the model adapter
**When** the Signal adapter sends it
**Then** Jay receives the response in Signal

**Given** a message arrives from a non-allowlisted number
**When** it reaches the Signal adapter
**Then** it is dropped with a log entry (not queued)

### Story 1.7: Claude Adapter Implementation

As the orchestrator,
I want to invoke Claude Code via CLI and parse responses,
So that Canti can think and respond.

**Acceptance Criteria:**

**Given** a valid session UUID exists
**When** a message is sent to the Claude adapter
**Then** it invokes `claude -p --resume <uuid> --output-format json`
**And** the JSON response is parsed for text + metrics
**And** response metrics (tokens, cache, context window) are extracted

**Given** no session exists
**When** the first message arrives
**Then** the adapter creates a new session with the system prompt
**And** the session UUID is persisted to disk

### Story 1.8: Session Manager (Basic)

As the orchestrator,
I want to maintain a singleton session,
So that Canti has one continuous identity.

**Acceptance Criteria:**

**Given** RustyClaw starts fresh
**When** the first message arrives
**Then** a new Claude session is created
**And** the session UUID is written to disk

**Given** RustyClaw restarts
**When** a persisted session UUID exists on disk
**Then** the session is resumed (not recreated)
**And** context from the previous conversation is retained

### Story 1.9: System Prompt Loading

As the orchestrator,
I want to construct the system prompt from files,
So that Canti's identity is declarative and versionable.

**Acceptance Criteria:**

**Given** config specifies prompt files (system.md, USER.md, etc.)
**When** a new session is created
**Then** the system prompt is assembled from those files in order
**And** the assembled prompt matches current Canti personality

### Story 1.10: End-to-End Integration Test

As a developer,
I want an integration test of the full message flow,
So that we can validate the MVP works before deploying.

**Acceptance Criteria:**

**Given** RustyClaw is running with mock Signal + real Claude adapters
**When** a test message is injected
**Then** it flows through queue → Claude → response
**And** the response is contextually appropriate
**And** session resume works on subsequent messages

---

## Epic 2: Session Lifecycle

**Goal:** Context management and memory persistence work reliably. Zero data loss.

**Success Criteria:**
- No conversation context lost during rotation
- Memory files updated before every rotation
- Context usage never exceeds hard threshold

### Story 2.1: Context Usage Monitoring

As the memory manager,
I want to track context usage from every Claude response,
So that I know when to trigger rotation.

**Acceptance Criteria:**

**Given** Claude returns response metrics
**When** the response is parsed
**Then** context_used_pct is calculated (total / contextWindow)
**And** the value is logged and stored in the metrics buffer

### Story 2.2: Pre-Rotation Memory Flush

As the memory manager,
I want to guarantee memory is written before rotation,
So that no conversation context is ever lost.

**Acceptance Criteria:**

**Given** a rotation is about to happen
**When** the pre-rotation flush is triggered
**Then** Claude is instructed to write important context to memory files
**And** the flush completes before the session is rotated
**And** if the flush fails, rotation is aborted with an alert

### Story 2.3: Session Rotation

As the session manager,
I want to start a fresh session when thresholds are exceeded,
So that context limits don't cause degradation.

**Acceptance Criteria:**

**Given** context_used_pct exceeds hard_threshold (configurable)
**When** rotation is triggered
**Then** pre-rotation flush completes first (Story 2.2)
**And** a summary of the conversation is generated
**And** a new session is created with the summary injected
**And** the old session UUID is archived
**And** the new session UUID is persisted to disk

### Story 2.4: Daily Session Rotation

As the scheduler,
I want to rotate sessions on a daily schedule,
So that sessions don't grow unbounded even in low-usage periods.

**Acceptance Criteria:**

**Given** a daily rotation time is configured (e.g., 4:00 AM)
**When** the time arrives
**Then** rotation follows the same flow as Story 2.3
**And** idle sessions (no messages in N hours) also trigger rotation

### Story 2.5: Soft Threshold Behavior

As the memory manager,
I want to write more aggressively when approaching limits,
So that important context is captured before it's too late.

**Acceptance Criteria:**

**Given** context_used_pct exceeds soft_threshold (configurable)
**When** each subsequent Claude response arrives
**Then** key context is logged to daily memory files more frequently
**And** a warning is logged for observability

---

## Epic 3: Scheduling & Observability

**Goal:** Cron jobs and monitoring work without OpenClaw.

**Success Criteria:**
- All 12 cron jobs fire on schedule
- Structured logs are queryable
- Health endpoint returns meaningful status

### Story 3.1: Cron Scheduler

As the orchestrator,
I want to execute scheduled tasks via the message queue,
So that MAKER, backups, and reports run automatically.

**Acceptance Criteria:**

**Given** schedules are defined in YAML config
**When** a cron expression fires
**Then** the task is enqueued at the configured priority
**And** it flows through the same message queue as user messages
**And** active conversations are not interrupted by low-priority cron jobs

### Story 3.2: One-Shot Timers (Reminders)

As a user,
I want to set reminders that fire once,
So that Canti can remind me of things.

**Acceptance Criteria:**

**Given** a reminder is set with a future timestamp
**When** the time arrives
**Then** the reminder message is enqueued
**And** it persists across RustyClaw restarts (persisted to disk)

### Story 3.3: Migrate 12 Existing Cron Jobs

As an operator,
I want all current OpenClaw cron jobs running in RustyClaw,
So that nothing breaks during migration.

**Acceptance Criteria:**

**Given** the 12 existing cron jobs are defined in RustyClaw config
**When** RustyClaw is running
**Then** all 12 fire on their existing schedules
**And** output matches OpenClaw's behavior for each job

Jobs: price-collection, coinglass-collection, research-nightly, maker-nightly-dd, nightly-backup, morning-report, gateway-config-cleanup, trash-reminder, weekly-allowance, market-data-refresh, petty-cash-reconciliation, anthropic-expense

### Story 3.4: Structured Logging

As an operator,
I want structured JSON logs,
So that I can debug issues and track behavior.

**Acceptance Criteria:**

**Given** any operation occurs in RustyClaw
**When** it is logged
**Then** the log entry is structured JSON with: timestamp, level, component, message, context
**And** log level is configurable per component
**And** logs rotate by size/date

### Story 3.5: Metrics Collection

As OPT (the self-tuner),
I want a rolling buffer of response metrics,
So that I have data to optimize against.

**Acceptance Criteria:**

**Given** every Claude response includes metrics
**When** they are collected
**Then** the last N responses are buffered in memory
**And** the buffer is checkpointed to disk periodically
**And** metrics include: tokens, cache hits, latency, cost

### Story 3.6: Health Endpoint

As an operator,
I want an HTTP health endpoint,
So that monitoring can verify RustyClaw is alive.

**Acceptance Criteria:**

**Given** RustyClaw is running
**When** GET /health is called
**Then** it returns 200 with: uptime, session status, queue depth, last message time
**And** if any critical subsystem is down, it returns 503

---

## Epic 4: Prompt Optimization & OPT

**Goal:** Cache-efficient prompt construction and self-tuning.

### Story 4.1: Zone-Based Prompt Layout

As the prompt optimizer,
I want to construct prompts in cache-aware zones,
So that static content stays cached across turns.

**Acceptance Criteria:**

**Given** a system prompt and conversation history
**When** a prompt is constructed for Claude
**Then** Zone 1 (static header) contains system prompt + base skills + USER.md
**And** Zone 2 (dynamic skills) contains loaded skills in append-only order
**And** Zone 3 (conversation) contains message history
**And** cache hit rate improves compared to naive prompt construction

### Story 4.2: Skill Loading Strategies

As the prompt optimizer,
I want to load skills either inline or via rotation,
So that cache efficiency is preserved.

**Acceptance Criteria:**

**Given** a skill needs to be loaded
**When** the skill is < inline_threshold_tokens
**Then** it is injected inline into the conversation body

**Given** a skill is > inline_threshold_tokens
**When** sufficient context remains
**Then** the session is rotated with the skill promoted to Zone 2

### Story 4.3: OPT Implementation

As a self-tuning system,
I want to analyze metrics and adjust configuration,
So that prompt/cache performance improves over time.

**Acceptance Criteria:**

**Given** a metrics buffer with N response data points
**When** OPT runs on schedule
**Then** it reads current config + metrics
**And** reasons about adjustments to tunable parameters
**And** writes updated config to disk (triggering hot-reload)
**And** logs every change with reasoning to the OPT changelog
**And** no parameter changes exceed max_change_per_run bounds

### Story 4.4: Config Hot-Reload

As the config system,
I want to detect file changes and reload,
So that OPT adjustments and manual edits take effect without restart.

**Acceptance Criteria:**

**Given** a config file is modified on disk
**When** the file watcher detects the change
**Then** the new config is validated
**And** if valid, the live config is swapped atomically
**And** if invalid, the change is rejected with a log warning

---

## Epic 5: MCP Servers

**Goal:** Data services accessible to Claude Code via Model Context Protocol.

### Story 5.1: memory-search MCP Server

As Claude,
I want to search memory files semantically,
So that I can recall past conversations and decisions.

**Acceptance Criteria:**

**Given** the memory-search MCP server is running
**When** Claude calls `memory_search(query, max_results)`
**Then** it returns ranked snippets with file path + line numbers
**And** search quality matches the current Python + pgvector implementation

### Story 5.2: habit-dojo MCP Server

As Claude,
I want to search the skill registry,
So that I can load the right skill for any task.

**Acceptance Criteria:**

**Given** the habit-dojo MCP server is running
**When** Claude calls `skill_search(query, top_n)`
**Then** it returns ranked skill matches with metadata
**And** results match current Python skill-search quality

### Story 5.3: MCP Server Lifecycle

As the operator,
I want MCP servers to start/stop independently,
So that they don't couple to RustyClaw's lifecycle.

**Acceptance Criteria:**

**Given** MCP servers are defined as separate systemd services
**When** RustyClaw starts
**Then** it connects to already-running MCP servers
**And** if an MCP server is down, Claude gracefully degrades (no crash)

---

## Epic 6: Decommission OpenClaw

**Goal:** RustyClaw is the sole orchestrator. OpenClaw retired.

### Story 6.1: Parallel Run Validation

As an operator,
I want to run RustyClaw alongside OpenClaw,
So that we can validate feature parity.

**Acceptance Criteria:**

**Given** both systems are running (different Signal numbers or routing)
**When** Jay uses RustyClaw for 2+ weeks
**Then** a comparison checklist confirms feature parity
**And** Jay confirms "this is better"

### Story 6.2: OpenClaw Retirement

As an operator,
I want to cleanly decommission OpenClaw,
So that there's only one system running.

**Acceptance Criteria:**

**Given** parallel validation passes
**When** decommission is approved by Jay
**Then** OpenClaw gateway service is stopped and disabled
**And** OpenClaw fork is archived (not deleted)
**And** all documentation is updated to reference RustyClaw
**And** celebration occurs 🎉

---

## Estimated Effort

| Epic | Estimated Time | Dependencies |
|------|---------------|--------------|
| Epic 1: Signal Bridge MVP | 2-3 weeks | signal-cli, Claude CLI |
| Epic 2: Session Lifecycle | 1-2 weeks | Epic 1 |
| Epic 3: Scheduling & Observability | 1-2 weeks | Epic 2 |
| Epic 4: Prompt Optimization & OPT | 2-3 weeks | Epic 3 |
| Epic 5: MCP Servers | 2-3 weeks | Epic 1+ (parallel) |
| Epic 6: Decommission | 1 week | All epics |

**Total: 10-15 weeks** (with parallel work on MCP servers)
