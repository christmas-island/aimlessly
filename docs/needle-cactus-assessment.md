# Needle × Cactus — Aimlessly Integration Assessment

**Date:** 2026-05-13
**Source:** <https://github.com/cactus-compute/needle>, <https://github.com/cactus-compute/cactus>
**Context:** Design extract for Aimlessly at [aimlessly-design-extract.md](aimlessly-design-extract.md)

---

## What is Needle?

A 26M parameter distilled function-call model from Cactus Compute. Custom "Simple Attention Network" architecture — encoder-decoder with cross-attention, no FFN in the encoder. Trained on 200B tokens (pretrain) + 2B tokens (post-train on single-shot function call data).

**Key stats:**
- 6,000 tok/sec prefill, 1,200 tok/sec decode (on Cactus engine)
- Beats FunctionGemma-270m, Qwen-0.6B, Granite-350m, LFM2.5-350m on single-shot function calling
- Single-shot only — not conversational
- d=512, 8H/4KV heads, BPE vocab 8192, 8 decoder + 12 encoder layers
- Weights open on HuggingFace: `Cactus-Compute/needle`

## What is Cactus?

A low-latency AI inference engine designed for mobile/wearable devices. The production runtime for Needle.

**Key properties for Aimlessly:**
- **OpenAI-compatible API** — `localhost:PORT` is a drop-in replacement for any OpenAI provider
- **Parsed function calls** in responses — returns structured `function_calls: []` natively
- **Cloud fallback built in** — auto-routes to cloud when local model can't handle the request
- **Rust SDK, Python SDK, C core** — flexible wrapping options
- **Tiny footprint** — LFM-1.2B runs in 76MB RAM on Mac M4. Needle at 26M would be ~20-30MB
- **brew install + `cactus run`** — designed to run as a standalone service

---

## Fit Assessment: Needle for Aimlessly

### Direct matches

**Section 1 — Synthetic Tools as Decision Opcodes**
Aimlessly's core primitive is synthetic tools that collapse LLM output into a structured decision. Needle was trained for exactly this — given a query + tool schemas, pick the right tool + arguments. The tool doesn't execute; it's just a decision opcode. Needle is purpose-built for this pattern.

**Section 9 — Fresh Synthetic Conversations Per Hop**
Each Aimlessly step constructs a fresh, tiny API call with a micro-prompt + synthetic tools. The design doc says:

> "Does this branch have uncommitted changes?" is a yes/no a tiny model can nail with a 500-token context window. You're not paying Opus prices for that.

Needle running locally makes these ~80% of graph steps essentially free — no API call, no latency, no cost.

### Not a fit

- **Meta-inspector calls (Section 3)** — needs multi-step reasoning about the graph itself, beyond single-shot tool calling
- **Real tool output interpretation (Section 7)** — one-hop cycles with large CLI/API output need more reasoning capacity
- **AI-edited graph construction (Section 4)** — compositional reasoning, not just tool selection

### Three-tier routing

Needle creates a clean routing model for the unresolved "model selection" gap in the schema:

| Tier | Model | Use case | Cost |
|------|-------|----------|------|
| **Local** | Needle via Cactus sidecar | Synthetic decision nodes, simple yes/no/route hops, context < 2K tokens | Free |
| **Mid cloud** | Gemini Flash, etc. | Tool result interpretation, moderate reasoning, real-tool one-hop cycles | Low |
| **Heavy cloud** | Opus, GPT-5, etc. | Meta-inspector, graph editing, complex multi-variable decisions | High |

The graph IS the routing logic. Each `prompt_node` defaults to local/Needle and escalates when explicitly marked.

---

## Deployment: Cactus Sidecar

**Recommended architecture:** Run Cactus as a sidecar container in the same pod as Aimlessly.

```
┌─────────────────────────────────────────┐
│  Pod                                     │
│                                          │
│  ┌──────────────┐   ┌────────────────┐  │
│  │ Aimlessly     │──▶│ Cactus sidecar │  │
│  │ (Go binary)  │   │ localhost:8080  │  │
│  └──────────────┘   └────────────────┘  │
│         │                   │            │
│         │          Needle weights (26M)  │
│         │          ~20-30MB RAM          │
│         │                                │
│         └──▶ Cloud providers (mid/heavy) │
└─────────────────────────────────────────┘
```

**Why sidecar works:**
- Aimlessly's harness already dispatches to LLM providers via API. Pointing to `localhost:8080` instead of `api.openai.com` is a config change, not an architecture change.
- No CGo bridge, no Python runtime embedding, no custom Go inference engine needed.
- Cactus handles model loading, quantization, KV-cache, prefill optimization — Aimlessly just sends chat completions.

**Schema addition — `model_tier` on prompt_node:**

```
prompt_node
  id, workflow_id, body
  model_tier: local | mid | heavy   (default: local)
  model_override: string | null     (e.g. "needle", "gemini-flash", specific model)
```

This resolves the "model selection mechanism" gap identified in the design extract. The harness reads `model_tier` and routes accordingly — local hits the Cactus sidecar, mid/heavy hit configured cloud providers.

---

## Open Questions / Risks

1. **Needle weight format compatibility** — Needle was trained for Cactus, but weights may need conversion to Cactus's internal format. Cactus already supports `functiongemma-270m-it` with tool calling; Needle should slot in similarly.
2. **Single-shot limitation** — Needle only does one function call per inference. Aimlessly's design already assumes this (each hop is one decision), but if a prompt node ever needs multi-tool reasoning, Needle won't cover it.
3. **Model maturity** — authors explicitly note "small models can be finicky." Needs testing against Aimlessly's actual prompt patterns before relying on it in production.
4. **k8s ARM support** — Cactus is optimized for ARM (Apple Silicon, Snapdragon). If the cluster runs AMD64 nodes, need to verify Cactus builds/runs there or use ARM node pools.

---

## Action Items

- [ ] Test Needle against Aimlessly prompt patterns (install Cactus locally, load Needle weights, run synthetic tool-call benchmarks)
- [ ] Add `model_tier` + `model_override` to the Aimlessly schema design
- [ ] Prototype the provider abstraction in the harness (local sidecar vs cloud dispatch)
- [ ] Evaluate Cactus container image for k8s deployment
- [ ] Track Needle maturity — watch for stability improvements, multi-shot support, larger variants
