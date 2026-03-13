---
name: lane-queue
description: >
  Two-layer serial execution queue with per-session and global concurrency control.
  Handles messages arriving during active agent runs via four queue modes (collect, steer,
  followup, steer-backlog). Use when building agent runtimes that need ordered message
  processing with configurable concurrency bounds and overflow policies.
license: MIT
compatibility: "Any JavaScript/TypeScript runtime. No native modules, no platform-specific APIs. Node.js, Deno, Bun compatible."
metadata:
  author: "Stackpedia"
  version: "1.0.0"
  tags: "concurrency, queue, serial-execution, agent-runtime, message-handling"
  block-profile: |
    extractability: HIGH
    lethal-quartet:
      private-data-access: false
      untrusted-content-exposure: false
      external-communication: false
      persistent-memory: false
    security:
      cves: []
      risk-classes: []
      credential-handling: "none"
      recommended-scanner: "n/a"
    dependencies:
      extraction-chain: ["command-queue.ts", "lanes.ts"]
      breaks-without: []
      optional-enhancements: ["logging-interface", "metrics-hooks"]
    composability:
      tested-frameworks: []
      existing-extractions: []
      compatible-with-rfc-11919: false
    source:
      repo: "openclaw/openclaw"
      files: ["src/process/command-queue.ts", "src/agents/pi-embedded-runner/lanes.ts"]
      version-tested: "v2026.2.26"
---

# Lane Queue — Block Profile

## 1. What It Does

A pure-TypeScript concurrency control pattern that enforces serial execution of agent tasks by default. It uses two layers: a **per-session lane** (one active agent run per conversation) and a **global lane** (bounds total parallel model calls across all sessions). When a new message arrives while the agent is already running, four queue modes determine what happens: collect it for batch processing, steer the current run, queue it for the next turn, or both.

Think of it like Python's `asyncio.Queue` with two stacked levels — one per-conversation, one global — plus policies for what to do when work arrives while you're already busy. No framework does this.

## 2. Why It's Useful Outside This Repo

Every agent runtime faces the same problem: what happens when a user sends a message while the agent is mid-turn? Most frameworks either drop it, crash, or race-condition their way through it. The Lane Queue solves this with explicit, configurable policies.

- **Chatbots**: Users send follow-up messages before the bot finishes responding. Collect mode batches them; steer mode redirects the current run.
- **Batch pipelines**: Global lane concurrency caps prevent overloading API rate limits across parallel sessions.
- **Multi-agent systems**: Subagent lanes with independent concurrency (e.g., 8 parallel subagents but only 4 main chat lanes) prevent resource starvation.
- **Cron/scheduled agents**: Cron lane runs at concurrency 1, guaranteed serial, never collides with interactive chat.

No equivalent exists in LangGraph (parallel supersteps), CrewAI (fixed sequential/hierarchical), AutoGen (actor model async), or Semantic Kernel (in-process orchestration). None of them handle "message arrives during active run" with explicit queue modes.

## 3. Key Files

| File | Role | Lines (approx) |
|---|---|---|
| `src/process/command-queue.ts` | Core queue: `enqueueCommandInLane()`, `drainLane()`, concurrency control, debounce | ~250 |
| `src/agents/pi-embedded-runner/lanes.ts` | `resolveGlobalLane()`, `CommandLane` enum (Main, Cron, Subagent, Nested) | ~60 |

## 4. External Dependencies

**None.** Zero npm packages. Pure TypeScript, Promises, and `Map`/`Set` from the standard library. Single-threaded, no worker threads, no external queue systems.

This is the cleanest dependency profile of any OpenClaw component.

## 5. Internal Coupling Points

Only two OpenClaw-specific imports need replacing on extraction:

| Import | What It Is | Extraction Cost |
|---|---|---|
| Logging helpers | 3 functions (`logInfo`, `logWarn`, `logError`) | Swap with any logger interface (console, pino, winston). ~10 minutes. |
| `CommandLane` enum | 4 string constants (`Main`, `Cron`, `Subagent`, `Nested`) | Inline or redefine as your own enum/union type. ~5 minutes. |

The queue itself is pure infrastructure. All OpenClaw business logic lives in the **consumers** that call `enqueueCommandInLane()` — the session manager, auto-reply pipeline, channel adapters, cron system. The queue doesn't know or care what it's queuing.

In Python terms: the queue is like a standalone `asyncio.Queue` subclass. The things that push work onto it (Flask routes, Discord handlers, cron jobs) are separate. Extracting the queue doesn't touch any of them.

## 6. Extraction Difficulty

**LOW.**

- Zero external dependencies to manage
- Two trivial imports to swap (logger + enum)
- No I/O, no network, no file access, no database
- No configuration system dependency (concurrency limits are constructor parameters)
- Could be extracted as a single file with the enum inlined

The original architecture doc rates this MEDIUM because it considers the session model coupling. But the session model is a *consumer*, not a dependency. The queue itself extracts cleanly.

## 7. What Breaks When You Extract It

**Nothing in the queue breaks.** The queue has no knowledge of OpenClaw's session model, gateway, or message format.

What you lose (and must reimplement in your consumer layer):

| Lost Feature | Why | Replacement Cost |
|---|---|---|
| Session-keyed lane creation | OpenClaw auto-creates a lane per session ID | Your runtime creates lanes keyed on whatever ID makes sense (conversation, user, thread) |
| Gateway message routing | OpenClaw's gateway decides which lane a message enters | Your message router calls `enqueueCommandInLane()` with the right lane key |
| Compaction triggers | OpenClaw triggers memory compaction via the lane queue | Only relevant if you're building a memory system — ignore otherwise |
| Queue mode selection | OpenClaw reads mode from config per-session | Pass mode as a parameter when enqueuing |

None of these are "breaks" — they're integration points your consumer code provides.

## 8. Existing Standalone Versions

**None.** This is confirmed across LangGraph, CrewAI, AutoGen, Semantic Kernel, and npm/PyPI searches.

- LangGraph uses parallel supersteps (different paradigm — graph-based, not queue-based)
- CrewAI has fixed sequential or hierarchical execution (no dynamic queue modes)
- AutoGen uses an actor model with async message passing (no serial-by-default guarantee)
- Semantic Kernel does in-process orchestration (no two-layer concurrency)
- Generic job queues (Bull, BullMQ, Celery) solve a different problem — they're for distributed work, not agent turn ordering

The Lane Queue pattern is novel. The "collect vs steer" vocabulary for handling mid-run messages doesn't exist anywhere else.

## 9. Lethal Quartet Assessment

| Element | Present? | Notes |
|---|---|---|
| Access to private data | **NO** | Queue processes opaque work items. It never reads message content, files, or credentials. |
| Exposure to untrusted content | **NO** | Queue doesn't parse, interpret, or route based on content. It's a FIFO with concurrency. |
| Ability to externally communicate | **NO** | No network calls, no HTTP, no WebSocket, no IPC. Pure in-process. |
| Persistent memory | **NO** | Queue state is ephemeral. Restarts clear it. No disk, no database. |

**Score: 0/4.** The Lane Queue is the **only** OpenClaw component that introduces none of the four risk elements. Every other component — memory, skills, browser, gateway, workspace config — carries at least one.

## 10. Security Considerations

### Known Vulnerabilities
None. Zero of OpenClaw's 9 CVEs follow this component. The Lane Queue has no network surface, no file access, no credential handling, and no input parsing.

### Inherent Risk Classes
None. The queue is security-neutral by design:
- No injection surface (doesn't process content)
- No escalation path (doesn't execute commands)
- No exfiltration channel (doesn't communicate externally)
- No persistence (state is ephemeral)

### Credential Handling
None. The queue never sees, stores, or transmits secrets. Work items are opaque — the queue manages ordering and concurrency, not content.

### What This Block Does NOT Protect Against
The Lane Queue provides **execution ordering**, not **security**. It does not:
- Validate what gets queued (your consumer code must validate inputs)
- Sandbox queued work (your executor must handle sandboxing)
- Rate-limit API calls (it limits concurrency, not request rate — different thing)
- Protect against malicious consumers (if attacker code can call `enqueueCommandInLane`, it can queue arbitrary work)

## 11. Recommended Mitigations

No security mitigations required for the queue itself. It's infrastructure plumbing with no attack surface.

For your **consumer code** that uses the queue:
- Validate work items before enqueuing (don't let untrusted input determine queue behavior)
- Set reasonable concurrency limits (unbounded concurrency = resource exhaustion)
- Set the overflow cap (OpenClaw defaults to 20 queued messages — choose a limit that fits your use case)
- Monitor queue depth in production (growing backlog = consumer can't keep up)

## 12. Tested Integration

*(To be filled after manual extraction and testing)*

## 13. Composability Notes

### Framework Integration Patterns

**LangGraph**: Implement as a custom node that wraps the LangGraph execution. Lane Queue sits *outside* the graph — it controls when graph runs start, not what happens inside them. Think of it as the bouncer at the door, not the bartender.

**CrewAI**: Replace CrewAI's built-in sequential/hierarchical process with Lane Queue for finer control. CrewAI's `Process.sequential` is a subset of what Lane Queue offers (it's like having only "followup" mode with concurrency 1).

**AutoGen**: AutoGen's `GroupChat` has no concept of "message arrived during active turn." Lane Queue fills this gap — wrap AutoGen agent execution in queued work items.

**Generic Python async**: Reimplement the pattern using `asyncio.Queue` + `asyncio.Semaphore`. The two-layer design (per-session + global) maps to nested semaphores. Queue modes map to different enqueue strategies.

### What Composes Well With Lane Queue
- Any message broker (Redis pub/sub, AMQP, WebSocket) as the input source
- Any LLM API as the work being queued
- Any logging/metrics system via the logger interface
- Memory systems (queue triggers memory operations, doesn't depend on them)

### What Doesn't Compose
- Distributed systems: Lane Queue is single-process. For multi-node, you need a distributed queue (Bull, Celery) and Lane Queue only manages the local node's concurrency.
- Streaming responses: Queue manages turn-level ordering. Within a turn, streaming is orthogonal — handle it in your executor.

## 14. Token Footprint

| Load Level | Approximate Tokens | When |
|---|---|---|
| Metadata only (YAML frontmatter) | ~150 tokens | Discovery/search results |
| This block profile (full) | ~2,200 tokens | Developer evaluating whether to use it |
| Source code (both files) | ~800 tokens | Implementing the pattern |
| Pattern description (for reimplementation) | ~500 tokens | Porting to another language |

The Lane Queue is one of the most token-efficient blocks to document and use. The entire pattern can be explained in under 500 tokens — small enough to include inline in an agent system prompt as an architectural reference.

---

## Extraction Recipe

For developers who want to use this pattern:

### Option A: Direct TypeScript Extraction (~15 minutes)
1. Copy `command-queue.ts` and the `CommandLane` enum from `lanes.ts`
2. Replace 3 logging imports with your logger (or `console.log`/`console.warn`/`console.error`)
3. Inline or redefine the `CommandLane` enum with your own lane names
4. Done. No package installs, no config, no build steps.

### Option B: Pattern Reimplementation (any language)
The core algorithm:
1. Maintain a `Map<string, Queue>` keyed on lane name
2. Each queue has a configurable concurrency limit (default: 1)
3. `enqueue(laneName, workFn)` adds to the lane's queue, starts draining if under concurrency limit
4. `drain(laneName)` pops work items and runs them up to the concurrency limit
5. Add a second layer: global lanes that bound total concurrency across all session lanes
6. Implement queue modes as enqueue strategies:
   - **collect**: buffer incoming, process all together when current work completes
   - **steer**: cancel pending tool calls in current work, inject new message
   - **followup**: append to queue, process after current work completes
7. Add debounce (default 1000ms) and overflow cap (default 20) for the collect buffer

### Key Design Decisions to Preserve
- **Serial by default.** Unconfigured lanes should default to concurrency 1. This is the opposite of most frameworks and it's intentional — reliability over speed.
- **Two layers, not one.** Per-session AND global. Without the global layer, N concurrent users each running at concurrency 1 still means N parallel API calls.
- **Queue modes are the innovation.** Without them, you just have a semaphore. The modes are what make this an agent-specific pattern rather than generic concurrency control.
