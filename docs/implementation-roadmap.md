# Aimlessly — Implementation Roadmap

**Date:** 2026-05-13
**Source:** Breakdown of [design-extract.md](design-extract.md) into implementable phases

---

## Phase 0: Schema & Data Model

**Goal:** Finalize exact tables/structs before writing engine code.

The design extract has a schema sketch with gaps. Resolve these first:

1. **Finalize `prompt_node` and `tool_node` schemas** — add `model_tier` and `model_override` fields (from [needle-cactus-assessment.md](needle-cactus-assessment.md))
2. **Resolve context injection placement** — edge-level (per-transition) or tool-level (per-tool). Recommendation: default to edge-level — more composable, same prompt gets different context from different incoming edges
3. **Define `edge` condition enum** — close it: `null | success | failure | param_match:value | timeout | stuck`
4. **Workflow metadata** — `confidence` update semantics (exponential moving average of success rate per 100 runs, updated by meta-inspector), `version` field for graph versioning
5. **Run record** — single execution: entry time, current node, call stack, step count, accumulated context, status (running/paused/completed/failed/stuck)
6. **Storage decision** — CockroachDB (match Allegedly)? SQLite (lighter for single-node)?

**Deliverable:** `schema.md` with exact table definitions + Go package with structs.

---

## Phase 1: Core Engine — Graph Walker

**Goal:** Walk a static graph end-to-end, making real LLM calls.

No editing, no meta-inspector, no inter-workflow calls. Just: given a workflow + entry context, walk prompt→tool→prompt→tool until a terminal node.

1. **Load graph from storage** — hydrate workflow nodes and edges into memory
2. **Provider interface** — abstract LLM dispatch: `Complete(messages, tools) → (tool_call, error)`. First implementation: OpenAI-compatible HTTP client (this is where Cactus sidecar plugs in later)
3. **Synthesize per-hop conversation** — prompt body + injected context + tool schemas → micro-prompt
4. **Dispatch to provider, parse tool call** — selected tool → find matching edge → traverse to next prompt node
5. **Context accumulator** — real-tool results get injected into next hop. Synthetic tools don't accumulate.
6. **Termination** — no outgoing edges, or `return` tool node
7. **Step limit** — hard cap (default 100) as safety rail before meta-inspector exists

**Deliverable:** `aimlessly walk <workflow-id> --input "..."` — CLI that runs a workflow end-to-end.

---

## Phase 2: Real Tools as One-Hop Cycles

**Goal:** Execute actual side effects (CLI commands, HTTP calls) within the graph walk.

1. **Tool kind dispatch** — non-synthetic tools (`cli`, `api`) pause the decision walk, execute, capture output
2. **CLI executor** — sandboxed subprocess with timeout, capture stdout/stderr/exit code
3. **API executor** — HTTP client from tool node definition, handle auth/headers
4. **Result injection** — tool output → context for follow-up prompt. Each real tool has an implicit follow-up prompt that interprets outcome and decides next edge
5. **Error handling** — failure → `failure` edge condition, success → `success`

**Deliverable:** Workflows that run shell commands and hit APIs mid-graph.

---

## Phase 3: Inter-Workflow Calls & Call Stack

**Goal:** `call` and `return` tool nodes — workflows compose like function calls.

1. **Call stack** — push `(workflow_id, node_id, context)` when hitting `call` node
2. **Workflow resolution** — look up target workflow by ID, jump to entry node
3. **Return handling** — `return` pops stack, carries result to caller's next edge
4. **Recursion guard** — max call depth (default 10). Hard limit until meta-inspector handles smart detection.
5. **Context scoping** — called workflow starts fresh, receives only explicit arguments from `call` node

**Deliverable:** Reusable sub-workflows. `git-commit` graph callable from any other graph.

---

## Phase 4: Meta-Inspector

**Goal:** Every-N-steps review that detects stuckness and proposes repairs.

1. **Trigger** — every N steps (configurable per workflow, default 20), walker pauses and invokes inspector
2. **Context synthesis** — last N tool choices, current prompt, accumulated context, step count, workflow confidence
3. **Inspector prompt** — special meta-prompt: "Is this agent stuck? If so, what's wrong and how should the graph be fixed?"
4. **Inspector tools** — synthetic: `continue`, `restart`, `skip` (jump to node), `edit` (propose graph modification)
5. **Edit proposal flow** — proposed edits go to `proposed_edit` table, not applied directly. Human review for now; auto-apply gated on confidence later.
6. **Confidence update** — inspector updates workflow confidence based on observed patterns

**Deliverable:** Self-monitoring workflows that detect and propose fixes for stuck states.

---

## Phase 5: AI-Editable Graphs (Construction)

**Goal:** A claw can build new workflows from scratch or modify existing ones.

1. **Graph construction API** — CRUD for workflows, nodes, edges. Go package + HTTP API.
2. **Construction prompt template** — meta-prompt taking task description → graph definition (JSON). Claw bootstraps new workflows via this.
3. **Trial-and-error loop** — construct → dry-run → inspect failures → modify → repeat
4. **Human-taught recording** — record (prompt, tool_call, result) triples from live sessions → distill into graph (later nice-to-have)
5. **Version history** — every edit creates new version. Rollback to any prior version. Confidence and run-count are per-version.

**Deliverable:** API where claws create, test, and iterate on workflows programmatically.

---

## Phase 6: Allegedly Coupling — Discovery Layer

**Goal:** Workflows discoverable through the knowledge graph.

1. **Workflow node type in Allegedly** — new node kind referencing Aimlessly workflow ID
2. **Discovery edges** — connect workflow node to concept nodes
3. **Discovery query** — "find workflow nodes connected to concepts matching <task>"
4. **Auto-registration** — new workflow → auto-create Allegedly node + edges from workflow description
5. **Actually integration** — index workflow metadata in Meilisearch for fast text search

**Deliverable:** Claws discover relevant workflows through the knowledge graph.

---

## Phase 7: Sensors as Graph Clients

**Goal:** Webhook/websocket/polling listeners trigger graph execution without custom code.

1. **Sensor harness** — generic listener: webhook receiver, websocket client, polling timer
2. **Sensor→graph binding** — source type, URL/config, target workflow ID, payload mapping
3. **Event→graph trigger** — event → new workflow run with payload as input context
4. **Graph-as-behavior** — parsing/relevance decisions are prompt nodes, not code. New source = new graph.
5. **Self-healing** — meta-inspector catches API payload changes, proposes graph edits to adapt

**Deliverable:** New integrations without new code — wire a sensor to a graph.

---

## Phase 8: Apparition (UI)

**Goal:** Visual editing and monitoring.

1. **Read views first** — workflow list, graph visualization (cycles, conditional edges), run history, execution state
2. **Write views second** — graph editor (add/move/delete nodes, edges, prompts, tool schemas)
3. **Cross-view navigation** — node → Allegedly knowledge → sensor triggers
4. **Build path** — Retool while schemas stabilize. Custom React when data model is frozen.

**Deliverable:** Usable UI for building and monitoring workflows.

---

## Dependency Chain

```
Phase 0: Schema
    ↓
Phase 1: Graph Walker
    ↓
Phase 2: Real Tools
    ↓
Phase 3: Inter-Workflow Calls
    ↓
Phase 4: Meta-Inspector
    ↓
Phase 5: AI-Editable Graphs
    ↓
Phase 6: Allegedly Coupling ←── can parallel with Phase 7
Phase 7: Sensors           ←── can parallel with Phase 6
    ↓
Phase 8: UI
```

**v0.1 scope:** Phases 0 → 1 → 2. Working graph executor that makes LLM calls and runs real tools. Everything else layers on top.
