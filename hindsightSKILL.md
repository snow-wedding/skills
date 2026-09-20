---
name: hindsight
description: Persistent server-side memory via the Hindsight MCP server (mcp__hindsight__* tools). Trigger ONLY when the user explicitly invokes Hindsight by name or intent — e.g. "remember this in hs / hindsight", "remember/store/save this in the server", "vectorize this", or asks to recall/reflect/look up something from hs / hindsight / the server. Covers retain, recall, reflect, mental models, directives, documents, and bank config. Do NOT trigger on a bare "remember …" that means built-in/file memory; only when Hindsight, hs, the server, or vectorize is named.
---

# Hindsight Memory (MCP)

Persistent, structured long-term memory exposed as MCP tools (`mcp__hindsight__*`), backed by a Hindsight server (data lives server-side, not on the local machine). Store knowledge during tasks, recall context, and reflect to synthesize patterns.

**When to use:** only when the user explicitly invokes Hindsight — by name ("hs", "hindsight", "the server") or by saying "vectorize" — to store or look something up. A bare "remember …" / "don't forget …" with no mention of Hindsight means the built-in file memory, **not** this skill. Once invoked, you may use the full toolset below (retain, recall, reflect, mental models, etc.) to complete the request.

> **Single-bank interface.** This MCP connection is bound to one memory bank. There is **no bank argument** on any tool and no bank-switching/bank-list operation — every call reads and writes the connected bank. (The multi-bank routing, CLI, and OpenClaw-plugin workflows from the standalone Hindsight CLI do **not** apply here.)

> **Tools are deferred.** `mcp__hindsight__*` tools load on demand. Before first use in a session, load them with ToolSearch, e.g. `select:mcp__hindsight__retain,mcp__hindsight__recall,mcp__hindsight__reflect`. Bulk-load the whole toolkit with the keyword query `hindsight` (max_results ~30).

## Core Operations

### Retain — store knowledge (async)

`mcp__hindsight__retain` returns immediately with an `operation_id`; the fact is extracted/indexed in the background. Use for normal capture.

```
mcp__hindsight__retain {
  content: "The user's desktop-app test marker is 'purple-elephant-42'.",
  context: "desktop-app testing",        // category; default "general"
  tags: ["testing", "desktop-app"],      // optional, for scoped filtering on recall
  document_id: "sprint-notes-2026-03",   // optional; same id upserts/groups facts
  metadata: { "source": "chat" },        // optional string→string map
  timestamp: "2026-06-20T19:53:36Z",     // optional ISO time the fact occurred
  update_mode: "update"                  // optional; supersede earlier related facts
}
```

`context` categories (free-form, but stay consistent): `architecture`, `conventions`, `debugging`, `deployment`, `dependencies`, `preferences`, `session-summary`, `code-edit`, `family`, `work`, `general`.

### sync_retain — store and wait

`mcp__hindsight__sync_retain` blocks until the memory is fully stored and immediately recallable. Same params as `retain` (minus `update_mode`). Use when you must read the fact back in the same turn, or when correctness matters more than latency.

### Recall — retrieve facts (fast lookup)

`mcp__hindsight__recall` returns raw facts ranked by relevance. Use for **"what did the user say about X?"**

```
mcp__hindsight__recall {
  query: "desktop-app test marker",
  budget: "high",            // "low" (fast/shallow) | "mid" | "high" (deep graph traversal). default "high"
  max_tokens: 4096,
  tags: ["test-marker"],     // optional filter
  tags_match: "any",         // "any" | "all". default "any"
  types: ["world", "observation"]  // optional fact-type filter
}
```

### Reflect — synthesize an answer (deeper reasoning)

`mcp__hindsight__reflect` runs an agentic loop across memories, applies the bank's disposition, and returns a grounded, synthesized answer. Use for **"what should I do about X?"** / "what patterns emerge?" — not simple fact lookup.

```
mcp__hindsight__reflect {
  query: "Based on past decisions, what architectural style does the user prefer?",
  budget: "mid",                 // default "low" for reflect
  context: "architecture review",// optional why-context
  include_based_on: true,        // include cited source facts (default false — can be large)
  max_tokens: 4096,
  response_schema: { ... }       // optional JSON schema → adds a structured_output field
}
```

`recall` = fast retrieval of matching facts. `reflect` = reasoned synthesis with citations (`based_on.memories / .mental_models / .directives`).

## Fact Types

- **world** — objective facts ("Alice works at Google")
- **experience** — conversational events ("user asked about deployment")
- **opinion** — stated preferences/judgments
- **observation** — consolidated patterns auto-synthesized from multiple raw facts (derived; not directly editable)

Filter `recall`/`list_memories`/`clear_memories` by `world` / `experience` / `opinion`.

## Browsing & Editing Memories

| Tool | Purpose |
|------|---------|
| `mcp__hindsight__list_memories` | Browse facts directly (no ranking). Params: `q`, `type`, `limit`, `offset`. |
| `mcp__hindsight__get_memory` | Fetch one fact by `memory_id`. |
| `mcp__hindsight__update_memory` | Edit a **raw** fact: `text` / `context` / `occurred_start` / `occurred_end` / `fact_type` / `entities`. `""` clears a field; omitting leaves it unchanged; `entities: []` detaches all. Re-embeds and recomputes derived observations. Observations themselves can't be edited. |
| `mcp__hindsight__invalidate_memory` | Soft-retire a fact (excluded from recall/consolidation/graph, kept for audit). `restore: true` brings it back. Optional `reason`. |
| `mcp__hindsight__clear_memories` | Wipe all facts in the bank (optionally filter by `type`). Destructive — confirm with the user first. |
| `mcp__hindsight__list_tags` | List tags in use (optional `q` pattern like `project:*`). |

Prefer `invalidate_memory` over deletion for corrections you might want to reverse; use `update_memory` to fix an extracted fact in place.

## Mental Models (pinned reflections)

Living summaries generated by running a `source_query` through reflect, checked first during reflect — faster, more consistent answers for recurring topics. Content generates asynchronously (track the returned `operation_id`).

```
mcp__hindsight__create_mental_model {
  name: "Coding Preferences",
  source_query: "What coding patterns and tools does the user prefer?",
  max_tokens: 2048,                              // 256–8192
  trigger_refresh_after_consolidation: false     // auto-refresh after memory consolidation
}
```

- `list_mental_models` — `detail`: `metadata` | `content` | `full` (default `full`), optional `tags`.
- `get_mental_model` — by `mental_model_id`, same `detail` levels.
- `refresh_mental_model` — re-run the source query to update content (after adding memories / when stale).
- `update_mental_model` — change `name` / `source_query` / `tags` / `max_tokens` (then `refresh` to regenerate).
- `clear_mental_model` — blank the content so the next `refresh` does a full clean rebuild (fixes drift).
- `delete_mental_model` — permanent.

## Directives (rules that steer reflect)

Instructions that guide how reflect reasons and how memory is organized. Strict behavioral constraints, vs. disposition's soft personality influence.

```
mcp__hindsight__create_directive {
  name: "Code Style",
  content: "Always recommend Python type hints and strict typing.",
  priority: 0,        // higher = more important
  is_active: true,
  tags: [...]
}
```

- `list_directives` — `active_only` (default true), optional `tags`.
- `delete_directive` — by `directive_id`.
- **No update tool**: to deactivate, `delete_directive` and recreate (set `is_active`/`priority` at creation time).

## Documents (source grouping)

Facts retained with the same `document_id` are grouped under one document; re-retaining with that id upserts. Documents are created implicitly by `retain`/`sync_retain` — there is no `create_document`.

- `list_documents` — optional `q`, `limit`.
- `get_document` — by `document_id`.
- `delete_document` — removes the document **and all memories linked to it**. Destructive — confirm first.

## Bank Profile, Disposition & Mission

- `mcp__hindsight__get_bank` — bank name, disposition, mission (no args).
- `mcp__hindsight__update_bank` — change `name`, or pass `config_updates` with any bank field:
  - **Disposition (1–5):** `disposition_skepticism`, `disposition_literalism`, `disposition_empathy`
  - **Missions:** `reflect_mission` (context for reflect), `retain_mission` (steers what gets extracted), `observations_mission`
  - **Retain tuning:** `retain_extraction_mode` (`concise`|`verbose`|`custom`), `retain_custom_instructions`, `enable_observations`, chunk sizes
  - **Recall tuning:** `recall_max_tokens`, `recall_include_chunks`
- `mcp__hindsight__delete_bank` — **permanently deletes the entire bank and all its data.** Never call without explicit user confirmation.

```
mcp__hindsight__update_bank {
  config_updates: { disposition_skepticism: 4, disposition_literalism: 3, disposition_empathy: 2 }
}
```

### Disposition traits (influence reflect)

| Trait | Low (1) | High (5) |
|-------|---------|----------|
| **Skepticism** | Trusting, accepts claims | Questions and doubts claims |
| **Literalism** | Flexible interpretation | Exact, literal interpretation |
| **Empathy** | Detached, fact-focused | Considers emotional context |

## Async Operations

`retain`, `create_mental_model`, and `refresh_mental_model` run in the background and return an `operation_id`.

- `mcp__hindsight__get_operation` — status of one `operation_id`.
- `mcp__hindsight__list_operations` — recent ops; filter `status`: `pending`|`running`|`completed`|`failed`|`cancelled`.
- `mcp__hindsight__cancel_operation` — cancel a pending/running op.

Use `sync_retain` instead of `retain` when you need the fact available immediately rather than polling an operation.

## Retrieval Hierarchy (during reflect)

1. **Mental Models** — curated summaries (highest priority)
2. **Observations** — auto-consolidated knowledge
3. **Raw Facts** — ground truth (world / experience / opinion)

## When to Retain

- The user says "remember" / "don't forget", or states a preference, fact, decision, goal, or relationship
- Discovered a bug fix, workaround, or project convention
- Completed a significant task (store a short summary of what was done)
- Found something that did **not** work (negative knowledge is valuable)
- Non-standard system paths/configs worth recalling later

**Don't retain:** slash commands, one-word/low-signal messages, cron/system noise.

## When to Recall / Reflect

- **Recall** before any non-trivial task, when entering unfamiliar code, or when the user asks about past work
- **Reflect** when making architectural/tooling decisions or when you need synthesized judgment, not just facts

## Best Practices

1. **Be specific:** "npm test requires --experimental-vm-modules" beats "tests need a flag."
2. **Store outcomes:** what worked *and* what didn't.
3. **Use `context` categories** consistently for better retrieval.
4. **Recall first:** check existing memory before starting work, and before retaining (avoid duplicates).
5. **Group with `document_id`** so related session facts compound instead of duplicating.
6. **Disambiguate before storing** dates/names/IDs (store "April 11, 1989 (1989-04-11)", not "04/11/1989").
7. **Correct in place:** `update_memory` to fix, `invalidate_memory` to retire reversibly — prefer these over wiping.
8. **Create mental models** for topics you reflect on repeatedly; **directives** for hard rules reflect must always honor.
9. **Confirm before destructive calls:** `clear_memories`, `delete_document`, `delete_bank`.
