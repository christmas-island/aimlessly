# Aimlessly — Consolidated Design Extract

This document consolidates every design idea, architecture concept, implementation detail, and naming decision related to Aimlessly (and its sibling libraries) from the Apr 20, 2026 design session and surrounding docs. The goal is a single reference doc Jake can use when writing the final spec before implementation.

**Source documents:**
- `2026-04-20-aimlessly-design.md` — main distilled design doc (Claude-authored extraction)
- `2026-04-20-aimless-naming-and-architecture.md` — raw conversation transcript (Jacob + Claude)
- `2026-04-20-naming-decisions.md` — settled names with rationale
- `2026-04-20-system-name-candidates.md` — bundle/system name candidates (still open)
- `2026-04-20-rejected-names.md` — rejected names (no design ideas found beyond naming)
- `knowledge-projects-index.md` — project index, full-stack context

---

## Settled Names

| Name | Component | Function |
|---|---|---|
| **Allegedly** | Knowledge graph library | Non-directional n-ary epistemological graph on CockroachDB. **Implemented.** |
| **Actually** | Meilisearch wrapper | Curated queryable knowledge surface over Allegedly. **Designed, not implemented.** |
| **Aimlessly** | Decision graph executor | Directed (non-acyclic) procedural graph engine. **Designed, not implemented.** |
| **Apparition** | UI | Unified view/edit interface across all stores. **Named only.** |

**Bundle/system name:** Still open. Candidates: Apocalyption (Jacob, post-conversation), Apocalypse, Automagical, Acceptable. Internal codename: Hive/Hivemind.

**Naming theme:** Epistemic-discourse adverbs / hedges, tongue-in-cheek. *"Allegedly, actually aimless."*

---

## 1. Core Primitive: Synthetic Tools as Decision Opcodes

**Origin:** Jacob (prior experiment, brought to the conversation)

The foundational insight: inject a micro-prompt with *fake* tools whose name + description + input schema force a deterministic, structured decision. Example: prompt is "decide if this conversation is directed at you and you should reply", tools are `reply` and `ignore`. The tools don't execute anything — they collapse the LLM's output space into a structured choice.

**Claude's framing:** This is a "prompt program" — a graph-structured executable where LLM calls are the transition function and synthetic tools are the opcodes. The tool just collapses output to a decision, which becomes a deterministic edge traversal.

**Why it works (Jacob):** Micro prompts + tiny context windows (a few thousand tokens) remove attention ambiguity. There's no room for the model to wander. The mechanism isn't "smaller models can do smaller tasks" — it's that "a small focused context removes the attention ambiguity that causes errors in the first place." This is a load-bearing reason the architecture works.

---

## 2. Graph Topology

**Origin:** Jacob (primary), Claude (extensions)

A directed graph, explicitly **not acyclic** (cycles are a feature, not a bug):

- **Prompt node** — contains stable instruction text (the "what to ask").
- **Tool node** — can be synthetic (decision-only), real (CLI/API side-effect), meta (self-referential graph editing), call (invoke another workflow), or return (pop call stack).
- **Edge** — connects nodes, carries conditions (null | success | failure | param_match:value).

Walking the graph: a prompt node gathers its attached tools, feeds the synthetic context to an AI, and whichever tool the AI selects determines the next prompt to walk to. Repeat.

**Claude's extension on edge conditions:** Edge conditions are where expressive power lives. Each edge from a tool node to a prompt node can carry conditions — tool succeeded, tool failed, parameter value matched X. This makes the whole structure a finite state machine where the LLM is the transition function. Loops fall out naturally (retry on failure, iterate until condition met) without needing a DAG constraint.

---

## 3. Cycle Handling via Meta-Inspector

**Origin:** Jacob (insight), Claude (analysis)

**The problem:** Non-DAG with LLM transitions means potential infinite loops. Algorithmic cycle detection is hard because two identical graph positions don't guarantee identical behavior (the LLM has context).

**Jacob's solution:** Every ~20 steps, an AI **meta-inspector** looks at the recent tool choices and decides whether the agent is stuck. If so, it's also a repair opportunity — the inspector can modify the graph to make it more resilient or fix the underlying issue. Cycle detection + on-the-fly learning in one mechanism.

**Claude's analysis of why this is elegant:**
- Sidesteps the halting problem — doesn't need to *prove* a cycle, just recognize "this looks stuck" the way a human would.
- The inspection step IS the repair step — clean separation of concerns. The stuck claw doesn't need to understand why; the inspector does.
- The inspector is a privileged moment where a potentially higher-capability or differently-prompted model reviews and edits the graph.

**Two distinct write paths into the graph:**
1. **Mid-execution self-edit** — the claw walking a workflow modifies it as it goes (should probably *propose* only).
2. **Meta-inspector edit** — the every-20-steps privileged review writes back from outside (validates and commits).

---

## 4. AI-Editable Graphs (Learning Loop)

**Origin:** Jacob (vision), Claude (mechanism)

Workflows should be **dynamic and editable by AI**. A claw can construct micro-prompts and tools, edit them until it has a repeatable, reliable workflow.

**Four construction paths (Jacob):**
1. **Human-taught** — sit with a claw, walk through a process once; it records the session as a graph.
2. **Trial and error** — claw constructs a graph from scratch for a novel task.
3. **Review and correction** — human inspects existing graph, edits.
4. **AI meta-review** — meta-inspector iterates.

**Claude's sustainable pattern:** humans seed, AI iterates. Doesn't require initial workflows to be perfect — just needs them to exist.

**Memory-systems analogy (Claude, possibly load-bearing):**
- Knowledge graph (Allegedly) ≈ **declarative memory** — what the system knows, flexible, editable.
- Decision graph (Aimlessly) ≈ **procedural memory** — what the system knows how to do.
- Brain synthesizing context per invocation ≈ **episodic recall**.
- Claw walking a workflow ≈ **habit / skill execution**.

**Implication:** research on procedural vs. declarative memory interaction in humans could inform how aggressively AI editing is allowed on mature vs. nascent workflows. Procedural memory is resistant to degradation; declarative is more flexible.

**Claude's concern:** A claw editing itself into incoherence is a real risk. Want versioning, or at least a proposed-edit → validation → commit flow.

**Joint conclusion:** meta-inspector is the natural home for consolidating edits. Mid-execution edits should propose; inspector validates and commits.

---

## 5. Data Model / Schema

**Origin:** Claude (sketched live), refined in conversation

```
workflow
  id, name, description, version, entry_node_id
  confidence (numeric — updated by meta-inspector based on observed
              success/failure; gates how aggressively edits are allowed)

prompt_node
  id, workflow_id, body (stable instruction text)

tool_node
  id, workflow_id
  kind: synthetic | cli | api | meta | return | call
  definition (JSON — tool schema for synthetic, command for cli,
              endpoint for api, target_workflow_id for call)
  context_injection (text or null — "important info" AI gets before prompt)

edge
  id
  from_node_id, from_node_type
  to_node_id, to_node_type
  condition (null | success | failure | param_match:value)
```

**Tool kinds explained:**
- `synthetic` — fake tool, decision opcode. Doesn't execute anything.
- `cli` — real shell/CLI invocation, treated as a one-hop side-effect cycle.
- `api` — real HTTP/RPC call, same one-hop cycle treatment.
- `meta` — self-referential: edit this graph, add a node, modify an edge. How the inspector and learning loop write back.
- `return` — pop the call stack, hand result to parent workflow.
- `call` — push current position onto call stack, transfer to target workflow's entry point.

---

## 6. Function-Call Semantics (Inter-Workflow Composition)

**Origin:** Jacob (insight), Claude (model)

Decision graphs can **call each other like function calls and return to the parent graph**. The same workflow gets reused across contexts:

- `git-commit` graph called from PR flow (commit → push → open PR)
- `git-commit` graph called standalone
- `git-commit` graph called from rebase flow

**Mechanism:** each workflow has an entry point and an exit tool — a `return` synthetic tool that carries a result back to the parent graph's call site. The call stack is just a stack of graph positions.

**Recursion is possible.** Claude: *"powerful or terrifying depending on the inspector's vigilance."*

---

## 7. Real Tools as One-Hop Cycles

**Origin:** Jacob (insight), Claude (cleaner formulation)

Real tool calls (CLI, API) act as **one-hop cycles** injected into the decision graph. Example: prompt is "find the document for Allegedly", tool is `list file tree at ..`, then follow-up prompt is "did we find the files and which are most relevant?"

**Claude's formulation:** A real tool call isn't really a decision node — it's a **side effect with an outcome**. It shouldn't be a first-class graph node; it should be a **sub-cycle**: execute the tool, inject the result as context, run a follow-up prompt that interprets the outcome and decides the next edge. The graph stays clean as a *decision* structure; real tools are context-enriching interrupts within a hop.

---

## 8. Stable Prompts + Injected Context

**Origin:** Jacob (insight), Claude (extension)

Prompts should **not be templatable** — too complex. Instead, two distinct sources of injected context:

1. **AI-given context** — the upstream tool node populates context based on what just happened (parameter values, real tool output).
2. **Dynamic, automatic memory additions** — the harness pulls relevant context from Allegedly/Actually based on the prompt's needs, automatically, without a hand-coded retrieval step.

**Clean separation:** the *what to ask* is fixed (prompt body), the *what you know when asking* is dynamic (injected context). Prompts can be reused across workflows with different injected context — more composable than templates.

**Open question (deferred by Jacob):** Does context injection live on the **edge** (per-transition — same prompt gets different context based on which tool transitioned in) or on the **tool node** (per-tool)? Jacob: *"a question to answer when we write the harness for agentic behavior."*

---

## 9. Execution Model: Fresh Synthetic Conversations Per Hop

**Origin:** Jacob (declaration)

**Key insight:** The Aimlessly harness is a **distinct agent type**, not a variant of existing claws.

**Mechanism:** Each prompt and its tools is used to construct a fresh synthetic conversation for an API call to an AI provider. When the harness sees a tool call used, it makes a **whole new** synthetic conversation for the next API call. No accumulated context drift between hops.

**Implications (Claude):**
- **No hallucination compounding.** Errors can't cascade across steps because there's no long session — just tiny unambiguous API calls.
- **Cost efficiency.** Most steps in a software workflow are embarrassingly simple when isolated. "Does this branch have uncommitted changes?" is a yes/no a tiny model can nail with a 500-token context window. You're not paying Opus prices for that.
- **Model selection bakes into node/workflow config** — not a meta-prompt. Cheap-fast for ~80% of steps, escalate at nodes that genuinely require heavier models. The graph IS the routing logic.

**Reframing of "agent":** The agent is the **harness** — the thing walking the graph, managing the call stack, assembling synthetic conversations, dispatching to providers, handling real tool execution. LLM instances are **stateless function calls**. Replaceable, cheap.

---

## 10. Coupling to Allegedly

**Origin:** Claude (question), Jacob (decision)

**Decision:** **Separate codebase and separate schema.** Allegedly's knowledge graph has structural assumptions that don't match this use case.

**But tightly coupled in a discovery sense:** knowing that a workflow exists is itself a piece of knowledge in the epistemological graph.

**Linkage model (Claude):** The knowledge graph holds a node representing each workflow (with a reference/ID into the procedural store), with edges connecting it to relevant concepts. Discovery flows through Allegedly ("I need to process an incoming message → I know there's a workflow for that"); execution happens in Aimlessly. The knowledge graph doesn't need to know the workflow internals — just that it exists and what it's for.

---

## 11. Workflow Granularity Is Emergent

**Origin:** Jacob (collapsed the question), Claude (why it works)

**Jacob:** The unit of work doesn't matter and shouldn't be pre-classified. A workflow might be "all the steps for a successful PR" or "how to answer a question from the knowledge graph" or "planning and executing software development across many days or weeks."

**Claude:** The graph structure itself encodes granularity. A "daily dev cycle" workflow is just a prompt node whose tool choices eventually walk into the "answer a question" workflow, which walks into smaller steps. Composition falls out of the edge structure — no need to decide granularity at design time.

**Unresolved implication:** The *durable vs. ephemeral* distinction probably still matters at the schema level. A workflow run 10,000 times with high success deserves different edit-protection than one scratched together five seconds ago. Likely connects to the `confidence` field on `workflow`.

---

## 12. Sensors as Aimlessly Clients

**Origin:** Jacob (insight), Claude (full closure)

"Sensor claws" (not actually claws) can be **directly wired to Aimlessly graphs**. New listener/websocket/polling/webhook integration types don't need custom code — they need a graph.

**Claude:** The *behavior* of a Sensor becomes **data, not code**. A new signal source doesn't require a new Sensor implementation; you teach the existing Sensor harness a new graph. Graph steps handle "is this event relevant?", "how do I parse this payload?", "what do I emit to Memory?" — all editable micro-prompts.

**Self-healing implication:** If a downstream API changes its payload shape, the meta-inspector notices parsing failures and modifies the graph to adapt — without a deploy.

---

## 13. Apparition (UI) Scope

**Origin:** Jacob (declaration), Claude (visualization needs)

Heterogeneous visualization needs:
- **Aimlessly graphs** — flowchart-style, must handle cycles, conditional edges, prompt vs. tool distinction, call/return stack visualization.
- **Allegedly** non-directional n-ary connections — force-directed layout.
- **Actually**-indexed Meilisearch docs — tabular, with relationship pointers back to graph nodes.
- **Sensor wiring** — pipeline/table view: which Sensor is bound to which Aimlessly entry point.

**Joint conclusion:** Unified UI with multiple views. Jumping from an Aimlessly node → the Allegedly knowledge backing it → the Sensor that fired it is worth the extra UI complexity.

**Build path:** Retool or similar early while schemas stabilize. Custom React app inevitable eventually. Don't build a beautiful graph editor for a schema that's going to change three times.

---

## 14. Implementation Language

**Origin:** Jacob

**Go**, to use Allegedly and Actually directly.

---

## Full Stack Summary

| Component | Role | Status |
|---|---|---|
| **Allegedly** | Knowledge graph (declarative memory) | Implemented (v6, all 70 issues closed) |
| **Actually** | Meilisearch wrapper (queryable knowledge surface) | Designed, not implemented |
| **Aimlessly** | Decision graph executor (procedural memory) | Designed Apr 20, not yet specified |
| **Apparition** | UI for all of the above | Named only |
| **Sensors** | Wired to Aimlessly graphs for code-free integrations | Reframed Apr 20 |
| **System bundle** | TBD — Apocalyption / Apocalypse / Automagical / Acceptable | Open |

---

## Contradictions & Unresolved Questions

### From the design doc (8 open questions):
1. **Context injection placement** — edge-level (per-transition) or tool-level (per-tool)? Jacob deferred to harness implementation.
2. **Versioning model for AI-edited graphs** — full versioned history, or proposed-edit-validation-then-commit, or both?
3. **Workflow node in Allegedly** — exact schema for the linking node and what edges it carries to concepts.
4. **Bootstrapping policy** — when does the Brain seed initial workflows vs. let claws construct them de novo?
5. **Recursion safety** — if `call` enables recursion, what prevents pathological self-recursive graphs from running unbounded? Inspector vigilance was the conversational answer; may need more.
6. **Mature vs. nascent workflow editing** — should AI editing be more conservative on high run-count / high success-rate workflows? The `confidence` field is the proposed lever.
7. **Durable vs. ephemeral workflows** — does the schema distinguish persistent learned procedures from one-shot scratch graphs, or does it fall out of confidence / run-count metrics?
8. **Two write paths** — mid-execution self-edits vs. meta-inspector edits. Same write API with proposal/commit semantics, or genuinely different paths?

### Contradictions / tensions identified:
- **Separate codebase but tight coupling.** The discovery linkage through Allegedly is clear conceptually but the exact integration mechanism (separate library? shared protobuf? direct DB queries?) isn't specified.
- **Mid-execution self-edits vs. inspector authority.** The conversation swung between "claws edit freely" and "mid-execution edits should propose, inspector commits." No clear resolution on where the authority boundary is.
- **No schema for confidence.** The `confidence` field is mentioned as gating edit aggressiveness but its update semantics (how is it computed? what triggers an update?) are undefined.
- **Model selection mechanism.** "The graph IS the routing logic" and "model choice bakes into node or workflow config" — but the schema sketch has no field for model selection. This is a gap.

### Additional details from the raw transcript not in the distillation:
- Jacob explicitly called this **"a new idea tonight"** — the design was spontaneous, not pre-planned. This means some decisions were made fast and may deserve more scrutiny.
- The conversation started with a failed attempt to use GitHub connector in Claude.ai, then pivoted to discussing knowledge graph context Jacob already had from prior conversations. The existing knowledge of the Allegedly/Actually architecture was a prerequisite for the Aimlessly design emerging.
- Jacob's initial framing included **"AI-generated 'next step' prompts"** — prompts that are dynamically generated rather than all being pre-authored in the graph. This idea was discussed but not fully resolved. The distillation focuses on stable prompts + injected context, but the raw transcript suggests some prompts could be generated on-the-fly by a prior step's LLM call.
- The `context_injection` field on `tool_node` in the schema sketch is somewhat at odds with the "context injection placement is open" question — the schema commits to tool-level, but the design question says edge-level is also possible.

---

## Attribution Summary

**Jacob-originated ideas:**
- Synthetic tools as structured decision outputs (prior experiment)
- Directed non-acyclic graph of prompts + tools
- AI-editable graphs / dynamic workflows
- Separate codebase from Allegedly
- Meta-inspector for cycle detection + repair (every ~20 steps)
- Workflow granularity is emergent (don't pre-classify)
- Function-call semantics for inter-workflow composition
- Real tools as one-hop cycles
- Prompts should not be templatable; use injected context instead
- Fresh synthetic conversations per hop (the harness is the agent)
- Go as implementation language
- Sensors wired directly to decision graphs
- Need a UI for graph editing

**Claude-originated ideas:**
- "Prompt program" framing (graph-structured executable)
- Edge conditions as the expressive power layer (FSM model)
- Memory-systems analogy (declarative vs. procedural vs. episodic)
- Linkage model (workflow node in Allegedly with discovery edges)
- Two write paths: mid-execution propose + inspector validate/commit
- Model selection as graph routing logic
- Cost analysis: 80% of steps can use cheap models
- Durable vs. ephemeral workflow distinction
- Confidence field on workflow schema
- All naming (Aimlessly, Apparition, and all candidates)

**Joint riffs:**
- The "Allegedly, actually aimless" tagline (Jacob spotted it, Claude ran with it)
- Self-healing integrations via meta-inspector modifying sensor graphs
- Build path: Retool early → custom React eventually
