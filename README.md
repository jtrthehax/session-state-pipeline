# LLM State Specification — SLS + DSR

**The missing layer in the modern LLM stack.**

[![DOI](https://zenodo.org/badge/1278357519.svg)](https://doi.org/10.5281/zenodo.20820098)

---

## The Problem

Every LLM session ends the same way: everything evaporates.

Not just the conversation — the *working context*. The task structure 
you built. The decisions you made and why. The hypotheses you were 
testing. The constraints you'd established. The momentum you had.

The model returns to a blank slate. You start over.

This is not a UX inconvenience. It is a structural architectural 
failure. Long-running workflows, multi-session reasoning, and 
collaborative agents are all blocked by the same root cause: 
**there is no standard for what model state is, how to capture it, 
or how to restore it.**

---

## The Solution

This repository defines two complementary specifications:

| Spec | Layer | Who It's For |
| --- | --- | --- |
| **SLS** — Serializable Latent State | Infrastructure | Self-hosted / provider-integrated deployments |
| **DSR** — Derived State Reconstruction | Application | Any hosted model (Claude, GPT-4, Copilot, etc.) |

**SLS** serializes the actual latent state of a running LLM — KV-cache 
tensors, sampling state, system bindings — into a portable, 
deterministic file. It enables resume, fork, migrate, and audit at the 
inference layer.

**DSR** achieves the same goal without infrastructure access. It defines 
a typed, human-readable **Structured State Envelope (SSE)** that 
captures what the model *knew* at session end — and can reconstruct 
it in a new session with 80–90% of native fidelity.

**SLS is the ideal. DSR is the deploy-today path.**

---

## Why DSR Is Not Just a Summary

Existing approaches fail in predictable ways:

| Approach | Problem |
| --- | --- |
| Full conversation replay | Exceeds context windows; poisons with stale content |
| Narrative summary | Lossy in unpredictable ways; not machine-parseable |
| KV-cache snapshot | Model-specific, opaque, gigabytes per session |

DSR is none of these. It applies a **semantic diff against model priors** 
— storing only what the model *cannot* regenerate from its training 
distribution. The result is a typed schema that is compact, 
inspectable, and model-agnostic.

The v0.2 spec adds three node types that no prior approach captures:

- **Instruction Nodes** — the procedural layer: what reasoning 
  operations were running, what lookups were pending, what checks 
  were flagged but not yet executed
- **Trajectory Nodes** — not just position but *direction*: where the 
  session was going, what paths were explicitly declined
- **Regulatory State Nodes** — the user's cognitive state at session 
  end: bandwidth, window width, and re-entry conditions

Together these enable the **Re-Entry Protocol** — a structured 
handshake that restores the *user*, not just the model, to the prior 
session's reasoning thread.

---

## Fidelity Levels

| Level | Name | Fidelity | Available Today |
| --- | --- | --- | --- |
| 0 | No state | 0% | Default for most LLMs |
| 1 | Preference memory | 5–15% | Copilot Memory, ChatGPT Memory |
| 2 | Session summary | 25–40% | Manual / OpenAI Agents SDK |
| 3 | Structured State Envelope | 50–75% | **This spec** |
| 4 | Augmented Envelope + excerpts | 70–90% | **This spec (advanced)** |
| 5 | Native KV-Cache (SLS) | ~100% | Requires infrastructure access |

Levels 3–4 are achievable today for any hosted model.  
Level 5 is the theoretical ceiling — SLS defines it.

---

## Repository Structure

```
/
├── README.md
├── specs/
│   ├── SLS_Initial_Proposal.md        ← KV-cache serialization spec
│   ├── SLS_Implementation_Guide.md    ← Engineering hooks + runtime
│   └── DSR_Specification.md           ← Semantic state envelopes (v0.2)
├── schemas/
│   └── sse.schema.json
├── examples/
│   ├── example_sse_level3.json
│   └── example_sse_level4.json
└── roadmap.md
```

**Start with DSR_Specification.md** if you are working with hosted 
models. Start with SLS_Initial_Proposal.md if you are working at the 
inference infrastructure layer.

---

## Status

| Document | Version | Status |
| --- | --- | --- |
| SLS Initial Proposal | v0.1 | Public draft |
| SLS Implementation Guide | v0.1 | Public draft |
| DSR Specification | v0.2 | Public draft — cognitive layer added |

---

## Related Work

The DSR spec's Regulatory State Nodes (5.8) and Re-Entry Protocol (Section 8) 
are grounded in a broader mechanistic model of human cognitive architecture — 
specifically, how prediction window width, autonomic state, and precision-gain 
interact to determine working bandwidth.

That model is documented separately:

**→ [Unified Model — Regulatory Architecture Framework](https://github.com/jtrthehax/Unified-Model)**

The Unified Model formalizes the biological substrate that DSR's user-facing 
layer is designed to interface with. If you want to understand *why* regulatory 
state is load-bearing at session boundaries — not just *that* it is — start there.

---

## Citation

```
Robinson, J. (2026). LLM State Specification: SLS + DSR.
Zenodo. https://doi.org/10.5281/zenodo.20820098
```

---

## License

MIT License — Copyright (c) 2026 Joel Robinson

