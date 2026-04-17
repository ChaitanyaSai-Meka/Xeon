# Xeon — System Design Document

> **Project Name:** Xeon  
> **Version:** v0.1 (Design Phase)  
> **Author:** Chaitanya Sai Meka  
> **Status:** Pre-implementation  

---

## Table of Contents

1. [Overview](#overview)
2. [Core Philosophy](#core-philosophy)
3. [System Boundaries](#system-boundaries)
4. [Foundation & State Management](#foundation--state-management)
5. [Execution Layer](#execution-layer)
6. [Governance Layer](#governance-layer)
7. [Dynamic Tool-Building Lifecycle](#dynamic-tool-building-lifecycle)
8. [Tool Validation Pipeline](#tool-validation-pipeline)
9. [Community Store](#community-store)
10. [Concurrency Model](#concurrency-model)
11. [Security Properties](#security-properties)
12. [Failure Handling & Crash Recovery](#failure-handling--crash-recovery)
13. [Multi-Language Support](#multi-language-support)
14. [Open Decisions & Known Risks](#open-decisions--known-risks)

---

## Overview

Xeon is a model-agnostic, locally-run autonomous agent system built in Go. Its defining characteristic is the strict architectural separation between **execution** (solving the user's problem) and **governance** (ensuring every action taken is safe and verified).

Unlike conventional agent frameworks that treat security as an afterthought, Xeon treats it as a first-class architectural concern. No custom code — whether written by a sub-agent or installed from the community — can ever execute without passing through an independent, multi-stage validation pipeline.

The system is designed to handle long-running, complex tasks that may require tools which do not yet exist, building and validating those tools dynamically at runtime.

---

## Core Philosophy

**Separation of Concerns** is the foundational design principle. The system is split into two fully independent domains:

| Domain | Role | Analogy |
|--------|------|---------|
| **Execution Layer** | Gets things done — planning, delegating, solving | The Government |
| **Governance Layer** | Keeps things safe — gating, auditing, approving | The Judiciary |

Neither domain has authority over the other's primary responsibility. The Execution Layer **cannot execute arbitrary code** — it must request permission through Governance. The Governance Layer **does not care about the user's goal** — it only cares about safety and correctness.

This separation means a compromised or hallucinating agent cannot unilaterally cause harm. Every action flows through a checkpoint it does not control.

---

## System Boundaries

```
┌─────────────────────────────────────────────────────────────┐
│                        XEON SYSTEM                          │
│                                                             │
│   ┌─────────────────────┐   ┌─────────────────────────┐   │
│   │   EXECUTION LAYER   │   │    GOVERNANCE LAYER      │   │
│   │                     │   │                          │   │
│   │  ┌───────────────┐  │   │  ┌──────────────────┐  │   │
│   │  │  Main Agent   │◄─┼───┼─►│  Command Center  │  │   │
│   │  │ (Orchestrator)│  │   │  │   (Gatekeeper)   │  │   │
│   │  └───────┬───────┘  │   │  └────────┬─────────┘  │   │
│   │          │          │   │           │             │   │
│   │  ┌───────▼───────┐  │   │  ┌────────▼─────────┐  │   │
│   │  │  Sub-Agents   │──┼───┼─►│   Judge Agent    │  │   │
│   │  │   (Workers)   │  │   │  │  (Zero-Trust LLM)│  │   │
│   │  └───────────────┘  │   │  └──────────────────┘  │   │
│   └─────────────────────┘   └─────────────────────────┘   │
│                                                             │
│   ┌───────────────────────────────────────────────────┐   │
│   │                  SHARED STATE                      │   │
│   │   instruction.md    memory.md    Tool Registry     │   │
│   └───────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## Foundation & State Management

### BYO Model (Bring Your Own Model)

Xeon is fully model-agnostic. Users configure their preferred LLM backend via a config file or environment variables. Supported options:

- **Cloud models** — OpenAI, Anthropic Claude, Google Gemini, etc. (via API keys)
- **Local models** — Ollama, LM Studio, or any OpenAI-compatible local endpoint for total privacy

The agent system does not care which model backs it. Each agent (Main, Sub, Judge) can be configured to use a different model independently, e.g., a cheap fast model for sub-agents and a smarter model for the Judge Agent.

---

### Flat-File Memory

All persistent state is stored in local Markdown files. This is an intentional design choice for transparency and debuggability — a user can open any file in a text editor and inspect the full state of the system at any point.

#### `instruction.md`
- Contains the user's global rules and behavioral constraints
- Written once by the user before a session (or updated between sessions)
- Read-only to all agents at runtime — no agent can modify this file
- Acts as the system's constitution — the Main Agent reads this before every task

#### `memory.md`
- The live scratchpad and context history
- Contains task progress, sub-agent reports, intermediate results, and decisions made
- **Single-writer rule** — only the Main Agent has write permission to this file
- Sub-agents cannot write to `memory.md` directly; they report findings to the Main Agent via the message queue, and the Main Agent decides what is worth persisting

This single-writer rule eliminates all file-level race conditions without requiring locks.

---

### Rolling Context Summarization

LLM context windows are finite. For long-running tasks, `memory.md` will eventually grow large enough to overflow the context window.

**Mechanism:**
- The Main Agent monitors the approximate token count of the active context
- Before the context window approaches its limit, the system triggers a summarization pass
- Older memory entries are compressed into a concise summary and replace the raw log
- The initial instructions from `instruction.md` are always re-injected at the top — the agent never forgets its core constraints regardless of how much history is compressed

This allows Xeon to run indefinitely on long tasks without losing continuity.

---

## Execution Layer

### Main Agent (The Orchestrator)

The brain of the system. The Main Agent is the single authoritative coordinator for the entire task lifecycle.

**Responsibilities:**
- Reads `instruction.md` on every task start
- Decomposes the user's prompt into a structured task plan
- Performs **upfront tool resolution** — before execution begins, lists all tools it anticipates needing
- Submits tool build requests to the Command Center in bulk
- Spawns and manages sub-agents for specific isolated work
- Is the **sole writer** to `memory.md`
- Acts as the routing hub for all sub-agent communications
- Decides whether to retry, escalate, or abort when a sub-agent fails

**What the Main Agent explicitly does NOT do:**
- Does not perform granular, detail-level work itself
- Does not execute any tool directly — all tool execution goes through the Command Center
- Does not spawn sub-agents to build more sub-agents — process tree is flat by design
- Does not write tool code itself

**Context hygiene:** The Main Agent only tracks high-level task status. It never loads raw sub-agent output directly into its context — sub-agents submit structured reports, keeping the orchestrator's context window clean.

---

### Sub-Agents (The Workers)

Ephemeral, task-scoped agents spawned by the Main Agent for specific, isolated pieces of work.

Examples of sub-agent tasks:
- "Write the CSS for this component"
- "Parse this JSON file and extract the schema"
- "Write the code for the `file_read` tool"

**Rules sub-agents operate under:**
1. Cannot spawn their own sub-agents — if a sub-agent determines it needs help, it reports back to the Main Agent, which decides whether to spawn additional workers
2. Cannot write to `memory.md` — all updates are communicated to the Main Agent
3. Cannot request tools directly — tool requests flow through the Main Agent → Command Center path
4. When writing a new tool, the sub-agent submits the code **directly to the Command Center**, bypassing the Main Agent. This keeps the Main Agent's context clean from intermediate code

---

### Kill Switch

Every sub-agent is wrapped in Go's `context.WithTimeout`.

```go
ctx, cancel := context.WithTimeout(parentCtx, maxSubAgentDuration)
defer cancel()
```

If a sub-agent enters a hallucination loop, stalls, or exceeds its time budget:
- The context is cancelled
- The sub-agent is immediately terminated
- The Main Agent receives a timeout error report
- The Main Agent decides: retry with a different prompt, spawn a fresh sub-agent, or escalate to the user

This prevents a single stuck sub-agent from blocking the entire system indefinitely.

---

## Governance Layer

### Command Center (The Gatekeeper)

The central authority over what code gets to exist and run in the system.

**Responsibilities:**
- Maintains the **Tool Registry** — the authoritative list of all approved, usable tools
- Receives tool build requests from the Main Agent
- Manages the full lifecycle of tool validation (static analysis → Judge → sandbox)
- Issues and tracks **tool leases** for each active task
- Performs **cleanup** when a task completes or crashes — expired leases are revoked and any task-specific tools are GC'd
- Acts as the delivery destination for tool code submitted by sub-agents

The Main Agent has no knowledge of how validation works — it simply requests a tool and either receives approval or rejection with feedback.

---

### Tool Leases

A tool built specifically for a task is not permanently added to the registry — it is issued as a **lease** tied to the originating task.

**Lease lifecycle:**
1. Task begins → Main Agent requests tools → Command Center issues leases
2. While task is running → leased tools are available for use
3. Task completes (success or failure) → Command Center automatically revokes leases and cleans up task-specific tools

**Lease constraints:**
- A leased tool cannot be used by other concurrent tasks unless it was independently approved for permanent registry inclusion
- Leases cannot be renewed infinitely — they are strictly scoped to one task

This prevents registry bloat and ensures that a tool written for one task's specific context doesn't accidentally influence an unrelated task.

---

### Judge Agent (The Zero-Trust Auditor)

A specialized LLM spawned by the Command Center for the sole purpose of security auditing and functional verification.

**Core properties:**
- **Zero-trust** — it has no context of the user's goal, the ongoing task, or any system history
- **Data-only** — submitted code is treated strictly as data. The Judge Agent's system prompt is engineered to be maximally resistant to prompt injection embedded inside tool code
- **Narrow scope** — its only job is to answer: "Is this code safe and does it do what it claims to do?"

**What the Judge Agent checks:**
- Logic flaws and incorrect implementations
- Infinite loops or unbounded recursion
- Subtle security risks not caught by static analysis
- Whether the tool's behavior matches its declared purpose
- Unexpected side effects

**What the Judge Agent ignores:**
- The user's overarching goal
- Whether the tool is useful
- Anything outside the submitted code

If the Judge Agent rejects a tool, it provides structured feedback. This feedback is passed back through the Command Center → Main Agent → original sub-agent chain for a retry with corrective guidance.

---

## Dynamic Tool-Building Lifecycle

This is the most architecturally complex part of Xeon, and its most differentiated capability.

### Step-by-Step Flow

```
1. UPFRONT RESOLUTION
   Main Agent analyzes task → lists all tools needed
   Checks Tool Registry for each
   Missing tools → bulk request to Command Center

2. LEASE ISSUANCE
   Command Center approves requests
   Issues task-scoped leases for each missing tool

3. PARALLEL BUILD (DAG-governed)
   Main Agent constructs a dependency DAG for the tool list
   Tools with no dependencies → goroutines launched in parallel
   Tools that depend on others → scheduled after dependencies complete

4. SUB-AGENT WRITES TOOL
   Each tool build task is assigned to a dedicated Sub-Agent
   Sub-Agent writes the tool code in the appropriate language

5. DIRECT HANDOFF TO COMMAND CENTER
   Sub-Agent submits code directly to the Command Center
   Main Agent is NOT in this loop — its context stays clean

6. VALIDATION PIPELINE (see next section)

7. GRANT OR REJECT
   Approved → tool written to registry, Main Agent notified it can proceed
   Rejected → structured feedback sent to Main Agent → retry or escalate
```

---

### Dependency DAG for Parallel Builds

When the Main Agent lists required tools upfront, it also declares dependencies between them.

Example:
```
Tool A: no dependencies      → starts immediately (goroutine)
Tool B: no dependencies      → starts immediately (goroutine)
Tool C: depends on Tool A    → starts after Tool A is approved
Tool D: depends on A and B   → starts after both are approved
```

This is structurally equivalent to a Makefile or a CI pipeline DAG. Go's concurrency primitives (goroutines + channels or `sync.WaitGroup` + error groups) are well-suited for this pattern.

**Cycle detection:** Before scheduling, the system checks the DAG for cycles. If a cycle is detected, the build is rejected and the user is notified — the Main Agent cannot resolve a circular tool dependency at runtime.

---

## Tool Validation Pipeline

Every piece of code — whether written by a sub-agent or installed from the community store — passes through the same multi-stage pipeline without exception.

```
Sub-Agent submits tool code to Command Center
              │
              ▼
┌─────────────────────────────────────┐
│  STAGE 1: AST Static Analysis       │
│  Language-specific pre-installed    │
│  analyzer for each supported lang   │
│                                     │
│  Checks:                            │
│  - Banned/dangerous imports         │
│  - Prohibited system calls          │
│  - Obvious malicious patterns       │
│                                     │
│  Cost: Near-zero (no LLM)           │
│  Speed: Milliseconds                │
└──────────────┬──────────────────────┘
               │ PASS
               ▼
┌─────────────────────────────────────┐
│  STAGE 2: Judge Agent (LLM Review)  │
│  Zero-trust LLM, code-as-data       │
│                                     │
│  Checks:                            │
│  - Logic flaws                      │
│  - Infinite loops / recursion       │
│  - Subtle security risks            │
│  - Intent vs implementation match   │
│  - Embedded prompt injection        │
│                                     │
│  Cost: One LLM call                 │
│  Speed: Seconds                     │
└──────────────┬──────────────────────┘
               │ PASS
               ▼
┌─────────────────────────────────────┐
│  STAGE 3: gVisor Sandbox Execution  │
│  Runtime behavioral verification    │
│                                     │
│  Mechanism:                         │
│  - Judge Agent generates mock       │
│    inputs based on tool signature   │
│  - Tool is executed under gVisor    │
│    (syscall-level isolation)        │
│  - Syscall trace is observed        │
│  - Output is compared to expected   │
│  - Timeout enforced — hang = fail   │
│                                     │
│  Cost: Container spin-up overhead   │
│  Speed: Seconds                     │
└──────────────┬──────────────────────┘
               │ PASS
               ▼
      Written to Tool Registry
      Main Agent granted use permission
```

**Why this order:**
- Stage 1 (AST) is free and fast — catches obvious violations before any LLM is invoked
- Stage 2 (Judge) is cheap and catches logical/subtle issues before runtime
- Stage 3 (gVisor) is the most expensive — reserved for code that has already cleared the first two gates

A tool must defeat all three stages independently. Defeating one stage is not sufficient.

---

### gVisor — Practical Notes

gVisor (`runsc`) intercepts syscalls at the kernel boundary, providing container-level isolation without requiring a full VM. It is the real enforcement boundary for runtime behavior.

**Known caveat:** gVisor does not implement every Linux syscall. Some Go runtime internals and certain C library functions behave unexpectedly under `runsc`. The Go toolchain should be tested against gVisor early in the implementation process — before the full pipeline is built around it — to confirm compatibility and identify any syscalls that need workarounds.

---

## Community Store

The community store allows users to install pre-built tools without writing them from scratch. It is a **distribution mechanism only** — the store does not grant any trust to its contents.

**Key properties:**
- Tools are not published to the store without prior screening (community store-level review)
- Despite this, every tool installed from the store still passes through the **full local validation pipeline** on the user's machine (AST → Judge → gVisor) before being written to the local registry
- "It passed the store review" is not a bypass condition — the store screening is defense in depth, not a replacement for local validation
- This makes the store resistant to supply chain attacks: even if a malicious tool passes store screening, it must still defeat the local pipeline

This is the same philosophy as package signing + local antivirus — two independent checkpoints, both must be satisfied.

---

## Concurrency Model

Xeon makes heavy use of Go's native concurrency primitives throughout.

### Main Agent Message Queue

The Main Agent is a coordination hub — sub-agents need to send it reports, memory update requests, and escalations. Under heavy parallel execution, naively having sub-agents call the Main Agent directly creates a bottleneck.

**Solution:** A buffered Go channel acts as the Main Agent's inbox.

```go
type AgentMessage struct {
    Type    MessageType // MEMORY_UPDATE | TOOL_REQUEST | ERROR | RESULT
    Payload any
    From    string // Sub-agent ID
}

mainAgentInbox := make(chan AgentMessage, bufferSize)
```

Sub-agents send messages to the channel and continue working. The Main Agent drains the channel at its own pace, processes requests, and dispatches responses. This decouples sub-agents from the Main Agent's current operation.

### Sub-Agent Lifecycle

Each sub-agent runs in its own goroutine with a context derived from the Main Agent's context:

```go
ctx, cancel := context.WithTimeout(mainCtx, subAgentTimeout)
defer cancel()
go runSubAgent(ctx, task, mainAgentInbox)
```

### Parallel Tool Builds

Tool builds are goroutines gated by a DAG scheduler. The scheduler uses a dependency-tracking structure — only goroutines whose dependencies have been satisfied (approved) are unblocked.

---

## Security Properties

The following security guarantees are upheld by the architecture:

| # | Property | Mechanism |
|---|----------|-----------|
| 1 | Main Agent cannot execute arbitrary code without independent validation | All tool use flows through Command Center → validation pipeline |
| 2 | Sub-agents cannot self-replicate | Sub-agents cannot spawn sub-agents; all spawning authority rests with Main Agent |
| 3 | No single point of failure in security | Two-stage validation (AST + gVisor) — a tool must defeat both independently |
| 4 | Judge Agent is prompt-injection resistant | Code treated strictly as data; Judge Agent has no task context; tightly scoped system prompt |
| 5 | Community tools receive zero implicit trust | Every installed tool runs the full local pipeline regardless of store screening |
| 6 | Process tree is flat and auditable | Maximum depth: Main Agent → Sub-Agent (never deeper) |
| 7 | Memory cannot be corrupted by sub-agents | Single-writer rule on `memory.md` |
| 8 | Lease expiry prevents registry pollution | Task-scoped tools are GC'd automatically on task completion or crash |

---

## Failure Handling & Crash Recovery

### Sub-Agent Timeout

- Triggered by `context.WithTimeout` expiry
- Main Agent receives a structured timeout error
- Main Agent decides: retry with refined prompt, spawn a replacement, or escalate to user

### Tool Validation Failure

- At any stage (AST, Judge, gVisor), rejection produces structured feedback
- Feedback is routed: Command Center → Main Agent → original Sub-Agent
- Sub-Agent retries with corrective guidance baked into its new prompt
- After N consecutive failures, the Main Agent escalates to the user rather than looping

### Task Crash / Abort

When a task terminates unexpectedly (user cancellation, unrecoverable error, system crash):
1. The Command Center is notified (or detects the crash via context cancellation)
2. All active leases tied to the crashed task are revoked
3. Task-specific tools are removed from the registry
4. `memory.md` is updated with the crash state (written by the Main Agent before exit, or via a recovery handler on restart)
5. On next startup, Xeon can inspect `memory.md` to determine whether to resume or start fresh

### Circular Tool Dependencies

- Detected before scheduling via DAG cycle check
- Not retried — reported to the user immediately as a task definition error

---

## Multi-Language Support

Tools can be written in the following languages, in priority order:

| Language | Priority | AST Analyzer | Notes |
|----------|----------|-------------|-------|
| **Go** | Primary | `go/ast` (stdlib) | Native to Xeon, always available |
| **Python** | Secondary | `ast` module (stdlib) | Pre-installed in the runtime environment |
| **Bash** | Tertiary | `shellcheck` | Pre-installed; best effort for dynamic constructs |
| **C** | Quaternary | libclang / Clang AST | Pre-installed; highest risk, most scrutinized |

**Language selection heuristic:** The Main Agent selects Go by default. Python is preferred when the task involves data processing or ML libraries. Bash is used for lightweight shell orchestration. C is reserved for performance-critical low-level tools.

**All languages pass through the same three-stage pipeline.** The AST analysis stage uses the appropriate language-specific analyzer, but Stage 2 (Judge Agent) and Stage 3 (gVisor) are language-agnostic.

---

## Open Decisions & Known Risks

These are architectural questions that are acknowledged but not yet finalized. They should be resolved before implementation begins on the affected components.

| # | Question | Options | Risk if deferred |
|---|----------|---------|-----------------|
| 1 | gVisor compatibility with Go runtime | Test early against `runsc` | Entire sandbox model may need to change if incompatible |
| 2 | Mock input generation for gVisor stage | Judge Agent generates inputs vs. hardcoded schema-based generation | Poor mocks = weak runtime verification |
| 3 | Max retry count for failed tool builds | Fixed N vs. user-configurable | Infinite retry loops possible if uncapped |
| 4 | Permanent vs. lease-only registry entries | Which tools graduate to permanent? | Registry grows unbounded if no graduation path |
| 5 | Memory schema versioning | How does `memory.md` format evolve across Xeon versions? | Old memory files may be unreadable after upgrades |
| 6 | Judge Agent model selection | Cheapest model vs. best model | Security-critical component — false negatives are costly |

---

*This document reflects the complete architecture as designed in the pre-implementation phase. It should be updated as implementation decisions are finalized.*
