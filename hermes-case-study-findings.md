# Hermes Case Study: Deep Findings, Concepts, and Reuse Guide

> **Purpose:** A deep, long‑form analysis of the Hermes Agent codebase for reuse in a memory‑ and empathy‑focused AI companion.  
> **Focus:** memory, user modeling, empathy‑oriented behaviors, and the supporting systems that make those reliable.

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Methodology and Sources](#methodology-and-sources)
3. [System Architecture Overview](#system-architecture-overview)
4. [Core Agent Loop and Tool Orchestration](#core-agent-loop-and-tool-orchestration)
5. [Memory System (Built‑in)](#memory-system-built-in)
6. [External Memory Providers (Pluggable Backends)](#external-memory-providers-pluggable-backends)
7. [Session Persistence and Long‑Term Recall](#session-persistence-and-long-term-recall)
8. [Prompt Assembly and Cache Stability](#prompt-assembly-and-cache-stability)
9. [Empathy and Persona Surface Area](#empathy-and-persona-surface-area)
10. [Skills System as Procedural Memory](#skills-system-as-procedural-memory)
11. [Toolsets, Tools, and Capability Gating](#toolsets-tools-and-capability-gating)
12. [Execution, Safety, and Guardrails](#execution-safety-and-guardrails)
13. [Gateway and Multi‑Platform Continuity](#gateway-and-multi-platform-continuity)
14. [UI/CLI/TUI Experience Layer](#uiclitui-experience-layer)
15. [Plugins and Extensibility](#plugins-and-extensibility)
16. [Cron, Delegation, and Parallel Work](#cron-delegation-and-parallel-work)
17. [Additional Features Worth Mining](#additional-features-worth-mining)
18. [Design Patterns to Reuse in Your Companion](#design-patterns-to-reuse-in-your-companion)
19. [Memory‑ and Empathy‑First Implementation Guidance](#memory--and-empathy-first-implementation-guidance)
20. [Risks, Tradeoffs, and Hard‑Won Lessons](#risks-tradeoffs-and-hard-won-lessons)
21. [Practical Adoption Roadmap](#practical-adoption-roadmap)
22. [Key File Reference Map](#key-file-reference-map)

---

## Executive Summary

Hermes is built around a **stable prompt core**, **bounded persistent memory**, and **tool‑first execution**. It adds **long‑term recall** via a SQLite session store and **pluggable memory providers** that can perform deeper, user‑modeling‑style recall. Empathy and persona are addressed through **SOUL.md identity**, **/personality overlays**, and a **background memory/skill review loop** that promotes user preferences into durable storage.

For a memory‑ and empathy‑focused companion, the most reusable concepts are:

- **Two‑tier memory**: lightweight local memory (short, curated) + external long‑term memory provider.  
  Files: `/home/runner/work/hermes-case-study/hermes-case-study/tools/memory_tool.py`,  
  `/home/runner/work/hermes-case-study/hermes-case-study/agent/memory_manager.py`.
- **User profile vs. environment memory**: separate USER and MEMORY channels with different semantics and limits.  
  Files: `/home/runner/work/hermes-case-study/hermes-case-study/tools/memory_tool.py`.
- **Frozen memory snapshot per session**: preserves prompt cache while still persisting updates to disk.  
  Docs: `/home/runner/work/hermes-case-study/hermes-case-study/website/docs/user-guide/features/memory.md`.
- **Background review for preference extraction**: a second agent pass to extract user preferences and skill improvements.  
  File: `/home/runner/work/hermes-case-study/hermes-case-study/agent/background_review.py`.
- **Session DB + FTS recall**: full conversation storage and zero‑LLM search across history.  
  File: `/home/runner/work/hermes-case-study/hermes-case-study/hermes_state.py`,  
  `/home/runner/work/hermes-case-study/hermes-case-study/tools/session_search_tool.py`.
- **Persona surface**: SOUL.md identity + optional personality overlays.  
  Files: `/home/runner/work/hermes-case-study/hermes-case-study/agent/prompt_builder.py`,  
  `/home/runner/work/hermes-case-study/hermes-case-study/hermes_cli/config.py`,  
  `/home/runner/work/hermes-case-study/hermes-case-study/hermes_cli/commands.py`.

---

## Methodology and Sources

### Path notation
All repository paths below are **absolute paths from this analysis environment**.  
Repo root used for those paths: `/home/runner/work/hermes-case-study/hermes-case-study`.  
If you are browsing the repo elsewhere, strip that prefix to get the repo‑relative path.

Primary sources referenced in this report (all paths are absolute, per repo context requirement):

- Core agent loop: `/home/runner/work/hermes-case-study/hermes-case-study/run_agent.py`
- Memory system:  
  `/home/runner/work/hermes-case-study/hermes-case-study/tools/memory_tool.py`  
  `/home/runner/work/hermes-case-study/hermes-case-study/agent/memory_manager.py`  
  `/home/runner/work/hermes-case-study/hermes-case-study/agent/memory_provider.py`
- Prompt assembly and persona identity:  
  `/home/runner/work/hermes-case-study/hermes-case-study/agent/system_prompt.py`  
  `/home/runner/work/hermes-case-study/hermes-case-study/agent/prompt_builder.py`
- Session DB + search:  
  `/home/runner/work/hermes-case-study/hermes-case-study/hermes_state.py`  
  `/home/runner/work/hermes-case-study/hermes-case-study/tools/session_search_tool.py`
- CLI personality configuration:  
  `/home/runner/work/hermes-case-study/hermes-case-study/hermes_cli/config.py`  
  `/home/runner/work/hermes-case-study/hermes-case-study/hermes_cli/commands.py`
- Toolsets and tools:  
  `/home/runner/work/hermes-case-study/hermes-case-study/toolsets.py`
- Docs (memory, sessions, prompt assembly, memory providers):  
  `/home/runner/work/hermes-case-study/hermes-case-study/website/docs/user-guide/features/memory.md`  
  `/home/runner/work/hermes-case-study/hermes-case-study/website/docs/user-guide/features/memory-providers.md`  
  `/home/runner/work/hermes-case-study/hermes-case-study/website/docs/user-guide/sessions.md`  
  `/home/runner/work/hermes-case-study/hermes-case-study/website/docs/developer-guide/prompt-assembly.md`  
  `/home/runner/work/hermes-case-study/hermes-case-study/website/docs/developer-guide/memory-provider-plugin.md`

---

## System Architecture Overview

At a high level, Hermes is organized as:

1. **Agent runtime (core loop)**  
   - The `AIAgent` in `/home/runner/work/hermes-case-study/hermes-case-study/run_agent.py` runs the conversation loop, handles tool calls, and manages per‑session state.
2. **Prompt assembly**  
   - Prompt structure is built in `/home/runner/work/hermes-case-study/hermes-case-study/agent/system_prompt.py` and `/home/runner/work/hermes-case-study/hermes-case-study/agent/prompt_builder.py`.
3. **Memory systems**  
   - Built‑in memory: `/home/runner/work/hermes-case-study/hermes-case-study/tools/memory_tool.py`  
   - Memory orchestration: `/home/runner/work/hermes-case-study/hermes-case-study/agent/memory_manager.py`  
   - Pluggable providers: `/home/runner/work/hermes-case-study/hermes-case-study/agent/memory_provider.py`
4. **Tools and toolsets**  
   - Tool registry and schemas: `/home/runner/work/hermes-case-study/hermes-case-study/tools/`  
   - Toolset definitions: `/home/runner/work/hermes-case-study/hermes-case-study/toolsets.py`
5. **Session persistence + search**  
   - DB: `/home/runner/work/hermes-case-study/hermes-case-study/hermes_state.py`  
   - Search tool: `/home/runner/work/hermes-case-study/hermes-case-study/tools/session_search_tool.py`
6. **UX layers (CLI/TUI/Gateway)**  
   - CLI: `/home/runner/work/hermes-case-study/hermes-case-study/cli.py`  
   - TUI: `/home/runner/work/hermes-case-study/hermes-case-study/ui-tui/` + `/home/runner/work/hermes-case-study/hermes-case-study/tui_gateway/`  
   - Gateway: `/home/runner/work/hermes-case-study/hermes-case-study/gateway/`

This is a **multi‑surface agent**: the same core loop can be surfaced in CLI, TUI, or messaging gateway sessions, with the memory and session DB spanning those interfaces.

---

## Core Agent Loop and Tool Orchestration

**Core entry point:** `/home/runner/work/hermes-case-study/hermes-case-study/run_agent.py`

Key behaviors:

- **Single system prompt per session:** built once and cached; only rebuilt on context compression or explicit reset.  
  This is fundamental to prompt‑cache efficiency and must be preserved if you emulate it.
- **Tool call routing:** tool schemas are gathered from core toolsets and from memory providers; tool calls are dispatched to handlers and results appended to the conversation.
- **Session DB integration:** session creation and message persistence happen through the session DB interface (`/home/runner/work/hermes-case-study/hermes-case-study/hermes_state.py`).
- **Memory provider lifecycle:** memory providers are initialized once, prefetched before turns, synced after turns, and invoked on session end.

**Why this matters for your companion:**  
If you want empathy and memory to be reliable, you must keep the agent’s system prompt stable across turns, while still persisting and updating memory behind the scenes. Hermes enforces that by *separating stable prompt components from live memory data*.

---

## Memory System (Built‑in)

**Implementation:** `/home/runner/work/hermes-case-study/hermes-case-study/tools/memory_tool.py`

### Two explicit memory stores
Hermes defines two bounded stores:

| Store | File | Purpose | Limit |
|---|---|---|---|
| MEMORY | `~/.hermes/memories/MEMORY.md` | Agent’s observations about environment and project | 2,200 chars |
| USER | `~/.hermes/memories/USER.md` | User profile/preferences/expectations | 1,375 chars |

These files are profile‑aware via `get_hermes_home()` in `/home/runner/work/hermes-case-study/hermes-case-study/hermes_constants.py`.

**Key design choices:**

- **Entries are delimited** by `§` and can be multiline.  
  This makes insertion and replacement cheap and simple.
- **Substring matching** for replace/remove encourages minimal operations instead of full text rewrite.
- **Duplicate prevention** automatically ignores identical entries.
- **Strict char limits** keep memory concise to minimize cost in every prompt.

### Frozen snapshot pattern (critical)
At session start, memory is loaded and **snapshotted** into the system prompt.  
During the session:

- The files are updated immediately (durable).
- The system prompt **does not change** (to preserve prefix caching).  

This is explained in `/home/runner/work/hermes-case-study/hermes-case-study/website/docs/user-guide/features/memory.md`.

This is a *non‑negotiable design choice* if you want efficient, repeatable responses without constant cache misses.

### Memory safety scanning
`/home/runner/work/hermes-case-study/hermes-case-study/tools/memory_tool.py` includes:

- **Injection/exfiltration scanning** before memory acceptance (e.g., “ignore previous instructions”, “curl $API_KEY”, `.env` reads).
- **Invisible Unicode detection** to block prompt‑injection via hidden characters.

For an empathy‑first companion, you should **keep this scanning**, because memory is injected into the system prompt and therefore can become a long‑term vector for prompt hijacking.

### Memory tool actions and write flow
The memory tool supports explicit actions with safety checks:

- **add** — append a new entry if it fits the limit and is non‑duplicate.
- **replace** — locate an existing entry by substring and replace it.
- **remove** — locate an existing entry by substring and remove it.

Important behaviors (all implemented in `/home/runner/work/hermes-case-study/hermes-case-study/tools/memory_tool.py`):

- **Substring targeting** allows small, human‑readable matches.
- **Conflict handling** rejects ambiguous matches (multiple entries).
- **Full state returned on error** to enable consolidation decisions when at capacity.
- **Atomic file replace + file locks** to avoid corruption across concurrent sessions.

### Memory context fencing and scrubbing
The memory manager introduces a special **memory‑context fence**:

- `build_memory_context_block()` wraps external provider recall in `<memory-context>` tags.  
  File: `/home/runner/work/hermes-case-study/hermes-case-study/agent/memory_manager.py`.
- `StreamingContextScrubber` strips those blocks from streamed output so the user does not see raw recall context.  
  File: `/home/runner/work/hermes-case-study/hermes-case-study/agent/memory_manager.py`.

This pattern lets the model see memory context **without leaking it** to the user interface, which is essential for privacy and UX clarity.

---

## External Memory Providers (Pluggable Backends)

**Provider ABI:** `/home/runner/work/hermes-case-study/hermes-case-study/agent/memory_provider.py`  
**Manager:** `/home/runner/work/hermes-case-study/hermes-case-study/agent/memory_manager.py`

### One external provider at a time
Hermes allows **only one external memory provider** at a time to avoid schema bloat and conflicting recall.  
The built‑in memory always remains active.

### Provider lifecycle hooks
Providers can implement:

- `initialize()` for session‑scoped setup.
- `system_prompt_block()` for static provider status in prompt.
- `prefetch()` / `queue_prefetch()` for recall context.
- `sync_turn()` for post‑turn persistence.
- `on_session_end()` for extraction/cleanup.
- `get_tool_schemas()` + `handle_tool_call()` for provider‑specific tools.

This interface is ideal if you want **advanced memory and empathy reasoning** without bloating your core agent loop.

### Provider list and characteristics
Detailed in `/home/runner/work/hermes-case-study/hermes-case-study/website/docs/user-guide/features/memory-providers.md`.  
Notably:

- **Honcho** emphasizes **user modeling** and dialectic reasoning.
- Others emphasize semantic recall, database persistence, or long‑term summarization.

If your product is empathy‑centric, Honcho‑style “user modeling + representation” is the closest direct match.

### MemoryManager per‑turn flow (what actually happens)
`/home/runner/work/hermes-case-study/hermes-case-study/agent/memory_manager.py` orchestrates:

1. **System prompt blocks**: `build_system_prompt()` pulls static provider blocks.
2. **Pre‑turn recall**: `prefetch_all()` merges provider recall into one context block.
3. **Post‑turn sync**: `sync_all()` persists the completed exchange.
4. **Next‑turn warmup**: `queue_prefetch_all()` starts background recall for the next turn.
5. **Session lifecycle hooks**: `on_session_end()` and `on_session_switch()` let providers flush or re‑scope state.

For your companion, this explicit sequencing is a template for mixing local memory + external providers in a predictable way.

### Provider selection and configuration
Provider selection is a **single‑select** model:

- Configuration key `memory.provider` in `~/.hermes/config.yaml`  
  (see `/home/runner/work/hermes-case-study/hermes-case-study/website/docs/user-guide/features/memory-providers.md`).
- Interactive setup via `hermes memory setup` (also described in the same doc).

This keeps provider‑specific APIs out of the core agent loop and lets you swap providers without touching core code.

---

## Session Persistence and Long‑Term Recall

**DB implementation:** `/home/runner/work/hermes-case-study/hermes-case-study/hermes_state.py`  
**Search tool:** `/home/runner/work/hermes-case-study/hermes-case-study/tools/session_search_tool.py`

### What Hermes stores
Hermes stores full session histories in SQLite with:

- All messages (user, assistant, tool calls/results)
- System prompt snapshot
- Session titles, timestamps, parent lineage
- Model metadata and token usage

This means **long‑term memory is not only short summary** — full transcripts are retained.

### Session search shapes
`session_search` provides:

1. **Discovery**: full‑text search with bookends.
2. **Scroll**: paginated navigation around a message id.
3. **Browse**: recent sessions listing.

This is a **no‑LLM recall path**; it avoids expensive model calls and provides precise recall.

### Empathy relevance
Session search is crucial for empathy: instead of shallow memory summaries, you can recall *verbatim* prior user moments or emotionally relevant context for better responses.

### Session lineage and WAL fallback
From `/home/runner/work/hermes-case-study/hermes-case-study/hermes_state.py`:

- **Parent session lineage** supports context compression and branching.  
  When a session is compressed or branched, the new session points to a parent; search tools resolve lineage to avoid duplicates.
- **WAL fallback behavior** detects NFS/SMB/FUSE environments and gracefully falls back from WAL to DELETE journal mode.  
  This avoids total failure on network filesystems and keeps session history usable.

These details matter if you want durable, portable memory across different host environments.

---

## Prompt Assembly and Cache Stability

**Assembly logic:**  
`/home/runner/work/hermes-case-study/hermes-case-study/agent/system_prompt.py`  
`/home/runner/work/hermes-case-study/hermes-case-study/agent/prompt_builder.py`  
`/home/runner/work/hermes-case-study/hermes-case-study/website/docs/developer-guide/prompt-assembly.md`

### Three‑tier structure
Hermes composes the system prompt into:

1. **Stable tier**  
   - Identity (`SOUL.md` or default identity)
   - Tool guidance and behavior rules
   - Skills index
   - Model‑specific operational guidance
2. **Context tier**  
   - Context files (`AGENTS.md`, `.hermes.md`, `.cursorrules`, etc.)
   - Optional system message
3. **Volatile tier**  
   - Memory snapshots (MEMORY + USER)
   - External memory provider block
   - Timestamp/session line

### Critical invariant: do not rewrite mid‑session
The system prompt is cached and reused across all turns.  
Only context compression rebuilds it. This protects prompt cache. For your companion:

- **Never mutate the cached prompt mid‑session** (unless you intentionally accept cache loss).
- Instead, update memory files and apply on next session boundary.

### Context file discovery and safety scanning
The prompt builder in `/home/runner/work/hermes-case-study/hermes-case-study/agent/prompt_builder.py` implements:

- **Context file priority**: `.hermes.md` / `HERMES.md` (walk to git root) → `AGENTS.md` → `CLAUDE.md` → `.cursorrules` / `.cursor/rules/*.mdc`.
- **Security scanning** for prompt injection (invisible Unicode, “ignore previous instructions”, exfiltration payloads).
- **Truncation** of overly long context files (head/tail with a marker).

This protects your prompt from poisoned project files and keeps context bounded.

---

## Empathy and Persona Surface Area

Empathy in Hermes is not a single module; it is **distributed across identity, memory, and review loops**.

### 1) SOUL.md identity (first‑class persona)
**File loader:** `/home/runner/work/hermes-case-study/hermes-case-study/agent/prompt_builder.py`  

SOUL.md is the *first* block in the system prompt when present.  
This is the place for:

- Voice and tone
- Compassion style
- Empathy or “companion” identity traits

If your product is empathy‑centric, treat SOUL.md as the canonical persona.

### 2) /personality overlays
Commands and config:  
`/home/runner/work/hermes-case-study/hermes-case-study/hermes_cli/commands.py`  
`/home/runner/work/hermes-case-study/hermes-case-study/hermes_cli/config.py`

This lets users change tone on demand (pirate, formal, etc.) without changing the baseline persona.  
It is not strictly empathy, but it gives **expressive flexibility**.

### 3) USER profile memory
USER.md is explicitly for user preferences, identity, communication style.  
This is where empathy‑critical details should land:

- Preferences for tone
- Emotional sensitivity triggers
- Boundaries or consent preferences

### 4) Background review loop
`/home/runner/work/hermes-case-study/hermes-case-study/agent/background_review.py` includes prompts that explicitly ask:

- Has the user revealed persona, desires, preferences?
- Has the user expressed expectations about behavior?

This is a **high‑signal empathy capture mechanism**.  
If you clone nothing else from Hermes, clone this *review prompt strategy*.

### Persona configuration knobs you can mirror
From `/home/runner/work/hermes-case-study/hermes-case-study/hermes_cli/config.py`:

- `display.personality` defaults to `"kawaii"` (UI persona selection).
- `agent.personalities` supports custom persona definitions (string or structured dict).

From `/home/runner/work/hermes-case-study/hermes-case-study/hermes_cli/commands.py`:

- `/personality <name>` sets a predefined personality overlay.
- CLI completions are built from `agent.personalities`.

These knobs separate **core identity** (SOUL.md) from **session‑scoped tone** (personality overlay).

---

## Skills System as Procedural Memory

Skills are declarative, stored instructions for “how to do things.”  
They are not just tools; they are **procedural memory**.

Key points:

- Skills are discovered and loaded from `/home/runner/work/hermes-case-study/hermes-case-study/skills/`.
- The agent can create/patch skills at runtime (`skill_manage`).
- Background review encourages skill updates whenever the user corrects style or workflow.

Empathy connection:  
If a user says “stop being verbose,” Hermes prefers to **update skills**, not just memory, so that the behavior is embedded into workflow guidance.  
This is a powerful pattern: **skills define how you behave**, memory defines who the user is.

---

## Toolsets, Tools, and Capability Gating

**Definition:** `/home/runner/work/hermes-case-study/hermes-case-study/toolsets.py`

### Core toolset
`_HERMES_CORE_TOOLS` includes web, terminal, files, memory, session_search, skills, browser automation, TTS, etc.  
This is the minimal cross‑platform tool palette.

At a high level (from `/home/runner/work/hermes-case-study/hermes-case-study/toolsets.py`), core categories include:

- Web: `web_search`, `web_extract`
- Terminal/process: `terminal`, `process`
- Files: `read_file`, `write_file`, `patch`, `search_files`
- Memory: `memory` + `session_search`
- Skills: `skills_list`, `skill_view`, `skill_manage`
- Browser automation: `browser_*` (navigate, snapshot, click, type, etc.)
- Vision and media: `vision_analyze`, `image_generate`
- Orchestration: `todo`, `delegate_task`, `execute_code`
- Scheduling: `cronjob`
- Cross‑platform messaging: `send_message`

This list is valuable when designing tool availability for empathy‑centric sessions.

### Why toolsets matter for empathy
You can restrict tools for sensitive contexts (e.g., disable shell execution for emotional support sessions).  
Tool gating is a safety + focus mechanism.

---

## Execution, Safety, and Guardrails

Hermes adds several safety patterns that matter for a companion:

1. **Memory content scanning** (see memory tool).  
2. **Context file scanning** to prevent prompt injection (`/home/runner/work/hermes-case-study/hermes-case-study/agent/prompt_builder.py`).  
3. **Approval workflows** for dangerous commands (present in `/home/runner/work/hermes-case-study/hermes-case-study/tools/approval.py`).  
4. **Tool use enforcement** in the system prompt (`/home/runner/work/hermes-case-study/hermes-case-study/agent/prompt_builder.py`).

Empathy systems can be undermined by prompt injection or unbounded tool calls. These guardrails are not optional if you plan to run untrusted user inputs through a long‑term memory system.

---

## Gateway and Multi‑Platform Continuity

The gateway (`/home/runner/work/hermes-case-study/hermes-case-study/gateway/`) provides:

- Unified conversation across Telegram, Discord, Slack, etc.
- Shared session IDs across platforms.
- The same memory and session DB used by CLI sessions.

For an empathy‑centric companion, this means:

- The AI identity and memory are consistent across channels.
- Users can engage in multiple contexts without losing continuity.

---

## UI/CLI/TUI Experience Layer

Hermes provides two user experiences:

1. **Classic CLI (prompt_toolkit)**: `/home/runner/work/hermes-case-study/hermes-case-study/cli.py`
2. **TUI (Ink / React)**: `/home/runner/work/hermes-case-study/hermes-case-study/ui-tui/` + `/home/runner/work/hermes-case-study/hermes-case-study/tui_gateway/`

This dual UI strategy allows:

- Rich, chat‑like UX (TUI)
- Minimal, terminal‑style interaction (CLI)

For your companion, you can adopt one of these models or build a separate UI that still uses the same underlying agent + memory.

---

## Plugins and Extensibility

Hermes has a plugin ecosystem:

- General plugins (tools, hooks): `/home/runner/work/hermes-case-study/hermes-case-study/plugins/`
- Memory providers: `/home/runner/work/hermes-case-study/hermes-case-study/plugins/memory/`
- Model providers: `/home/runner/work/hermes-case-study/hermes-case-study/plugins/model-providers/`

**Memory providers are a core extension point** for empathy‑focused capabilities.

### Memory provider plugin structure (how to reuse)
From `/home/runner/work/hermes-case-study/hermes-case-study/website/docs/developer-guide/memory-provider-plugin.md`:

```text
plugins/memory/<name>/
├── __init__.py      # MemoryProvider implementation
├── plugin.yaml      # Metadata (name, description, hooks)
└── README.md        # Setup + config reference
```

For your system, this is a clean boundary: memory providers can evolve independently and even be swapped per user profile.

---

## Cron, Delegation, and Parallel Work

These systems are not strictly empathy‑related but useful for assistants that:

- Perform scheduled check‑ins (cron).
- Delegate tasks to subagents for complex research.

They live in:

- `/home/runner/work/hermes-case-study/hermes-case-study/cron/`
- `/home/runner/work/hermes-case-study/hermes-case-study/tools/delegate_tool.py`

---

## Additional Features Worth Mining

These are not strictly empathy‑centric, but they improve usability and reliability:

- **Context compression engine**: `/home/runner/work/hermes-case-study/hermes-case-study/agent/context_compressor.py`  
  Keeps long sessions usable; emits summaries without rewriting the base prompt.
- **Tool registry + schemas**: `/home/runner/work/hermes-case-study/hermes-case-study/tools/registry.py`  
  Centralizes tool discovery and schema definitions for the model.
- **Checkpointing** (filesystem snapshots): surfaced in CLI flags and run loop for safer edits.  
  Primary loop in `/home/runner/work/hermes-case-study/hermes-case-study/run_agent.py`.
- **Background review for skills**: the same mechanism that saves memory also patches skills.  
  File: `/home/runner/work/hermes-case-study/hermes-case-study/agent/background_review.py`.

These can raise the quality of a companion even if your focus is empathy.

---

## Design Patterns to Reuse in Your Companion

### Pattern 1: Frozen memory snapshot + live memory writes
**Why:** preserves prompt cache while still persisting memory updates.  
**Where:** `/home/runner/work/hermes-case-study/hermes-case-study/tools/memory_tool.py`  
**Reuse:** snapshot at session start; allow writes to update storage; refresh on next session boundary.

### Pattern 2: Two‑channel memory model
**Why:** user preferences and environment facts are distinct, but both matter.  
**Where:** `/home/runner/work/hermes-case-study/hermes-case-study/tools/memory_tool.py`  
**Reuse:** USER.md (user identity & preferences) vs MEMORY.md (project/environment).

### Pattern 3: Background review agent for preference extraction
**Why:** real‑time agent responses miss subtle preference shifts.  
**Where:** `/home/runner/work/hermes-case-study/hermes-case-study/agent/background_review.py`  
**Reuse:** run a low‑cost background model after each turn; ask “what should be remembered?”

### Pattern 4: Session search as “memory substrate”
**Why:** avoid summary distortion, allow verbatim recall.  
**Where:** `/home/runner/work/hermes-case-study/hermes-case-study/hermes_state.py`, `/home/runner/work/hermes-case-study/hermes-case-study/tools/session_search_tool.py`  
**Reuse:** keep full conversation logs in local DB, searchable without LLM calls.

### Pattern 5: SOUL.md as primary identity layer
**Why:** clear, user‑editable persona baseline.  
**Where:** `/home/runner/work/hermes-case-study/hermes-case-study/agent/prompt_builder.py`  
**Reuse:** treat persona as a local, user‑editable artifact, injected at prompt start.

---

## Memory‑ and Empathy‑First Implementation Guidance

Below is a detailed, practical mapping of Hermes concepts to an empathy‑centric companion.

### A) Define memory tiers explicitly

**Hermes pattern:** `MEMORY.md` (environment) + `USER.md` (user profile).  

**Your adaptation:**

- **User Profile Store (USER):**
  - Preferences (tone, verbosity)
  - Sensitive boundaries (“avoid discussing X”)
  - Emotional support style (validation vs. problem‑solving)
  - Identity descriptors (name, pronouns, timezone)
- **Context Store (MEMORY):**
  - Environment facts
  - Ongoing project info
  - Tools/configurations

**Key rule:** keep user‑level emotional preferences in USER, not in general context.

### B) Use background review for empathy capture

Hermes explicitly asks “what should be saved about the user?” after each turn.  
Use this as an **automatic empathy extractor**:

1. Feed conversation transcript to a review model.
2. Ask it to extract **user emotional preferences** and **behavior expectations**.
3. Save to USER memory store.

This ensures the companion grows increasingly aligned with the user’s emotional style without constant explicit prompts.

### C) Preserve prompt cache while updating memory

Hermes’ frozen snapshot pattern is one of the most critical performance and correctness techniques:

- You can keep memory writes live and durable.
- You avoid re‑rendering a huge prompt every turn.

This yields:

1. **Stable behavior** (the model isn’t asked to “re‑learn” mid‑turn).
2. **Lower latency** (cache hits stay valid).

### D) Add external memory providers for deep recall

Use a provider interface similar to Hermes (`MemoryProvider`):

- Allows you to plug in user modeling or embedding‑based recall later.
- Avoids coupling your core loop to a single backend.

**Empathy use case:** external providers can store “latent” user representations, emotion trends, and long‑term relationship models.

### E) Separate “user identity” from “skills”

Hermes treats skills as **how‑to knowledge**, not user identity.  
For your system:

- Use **skills** for interaction style for *classes* of tasks.
- Use **user memory** for identity and emotional preferences.

This prevents your system from overfitting to a single user’s preferences across all tasks.

---

## Risks, Tradeoffs, and Hard‑Won Lessons

### 1) Prompt cache vs. memory freshness
Hermes chooses cache stability over mid‑session memory refresh.  
Tradeoff: the agent won’t immediately “see” new memory in its prompt.  
Mitigation: show tool results, and refresh on next session.

### 2) Memory bloat
Strict char limits keep memory “sharp.”  
If you let memory expand unbounded, your empathy‑model can become noisy and inconsistent.

### 3) Prompt injection through memory
Since memory is injected into the system prompt, it becomes a long‑term attack surface.  
Hermes scans for injection/exfiltration patterns.  
Keep this protection in any empathy‑first system.

### 4) User privacy and consent
Long‑term memory is sensitive.  
You should explicitly design:

- Clear memory controls (delete, inspect, opt‑out)
- Transparency about what is stored
- Limits on sensitive categories (especially for emotional data)

---

## Practical Adoption Roadmap

### Phase 1: Core Memory + Persona
1. Implement **SOUL.md‑style identity**.
2. Implement **USER + MEMORY stores** with char limits.
3. Implement **snapshot injection** at session start.

### Phase 2: Background Review Loop
1. Add post‑turn review prompting for user preferences.
2. Save extracted preferences to USER memory.
3. Introduce “Nothing to save” fallback to avoid noise.

### Phase 3: Session Storage + Search
1. Store full transcripts in SQLite (FTS5).
2. Add a “session_search” tool for recall.
3. Use session recall to avoid overloading memory with transcripts.

### Phase 4: External Memory Providers
1. Define a provider interface (initialize/prefetch/sync).
2. Build a plugin discovery path.
3. Add a user modeling backend (Honcho‑style or custom).

### Phase 5: Toolset Gating for Emotional Safety
1. Create toolsets for “support sessions” vs “developer mode.”
2. Disable risky tools during emotional contexts.

---

## Key File Reference Map

Use this as a quick navigation index for the most relevant artifacts.

### Memory
- `/home/runner/work/hermes-case-study/hermes-case-study/tools/memory_tool.py`
- `/home/runner/work/hermes-case-study/hermes-case-study/agent/memory_manager.py`
- `/home/runner/work/hermes-case-study/hermes-case-study/agent/memory_provider.py`
- `/home/runner/work/hermes-case-study/hermes-case-study/website/docs/user-guide/features/memory.md`
- `/home/runner/work/hermes-case-study/hermes-case-study/website/docs/user-guide/features/memory-providers.md`

### Persona and Prompt Assembly
- `/home/runner/work/hermes-case-study/hermes-case-study/agent/prompt_builder.py`
- `/home/runner/work/hermes-case-study/hermes-case-study/agent/system_prompt.py`
- `/home/runner/work/hermes-case-study/hermes-case-study/website/docs/developer-guide/prompt-assembly.md`
- `/home/runner/work/hermes-case-study/hermes-case-study/hermes_cli/config.py`
- `/home/runner/work/hermes-case-study/hermes-case-study/hermes_cli/commands.py`

### Sessions and Recall
- `/home/runner/work/hermes-case-study/hermes-case-study/hermes_state.py`
- `/home/runner/work/hermes-case-study/hermes-case-study/tools/session_search_tool.py`
- `/home/runner/work/hermes-case-study/hermes-case-study/website/docs/user-guide/sessions.md`

### Tools and Toolsets
- `/home/runner/work/hermes-case-study/hermes-case-study/toolsets.py`
- `/home/runner/work/hermes-case-study/hermes-case-study/tools/`

### Background Review
- `/home/runner/work/hermes-case-study/hermes-case-study/agent/background_review.py`

---

## Closing Notes

If your goal is a companion that feels empathic and remembers deeply:

- Build **memory first**, not tools.
- Use **system prompt stability** to keep behavior consistent.
- Employ **background review** to capture preferences you might miss in real time.
- Separate **user identity memory** from **procedural skills**.
- Use **session search** for rich recall instead of bloating memory.

If you want a follow‑up, I can tailor the next document to:

- A step‑by‑step implementation blueprint, or
- A specific subsystem (memory provider, persona scaffolding, review loop), or
- A comparative analysis against another companion framework.
