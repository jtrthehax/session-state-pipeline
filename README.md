# SSP — Session State Pipeline

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

SSP defines a **four-phase lifecycle** for AI session state:

**Extract → Consolidate → Persist → Restore**

Each phase is typed, deterministic, and model-agnostic. Each phase
is independently verifiable. The pipeline is implementable today
without provider cooperation.

The pipeline operates at two fidelity levels:

| Level | Track | Mechanism | Coverage |
| --- | --- | --- | --- |
| High | **SLS** — Serializable Latent State | Native KV-cache serialization | ~100% |
| Standard | **DSR** — Derived State Reconstruction | Prompt-based semantic extraction | 50–90% |

**SLS** serializes the actual latent state of a running LLM — KV-cache
tensors, sampling state, system bindings — into a portable, deterministic
file. It requires infrastructure access.

**DSR** achieves the same goal without infrastructure access. It defines
a typed, human-readable **Structured State Envelope (SSE)** that captures
what the model *knew* at session end.

**SLS is the ideal. DSR is the deploy-today path. SSP is the lifecycle
that connects them.**

---

## Why This Is Not Just a Summary

Existing approaches fail in predictable ways:

| Approach | Problem |
| --- | --- |
| Full conversation replay | Exceeds context windows; poisons with stale content |
| Narrative summary | Lossy in unpredictable ways; not machine-parseable |
| KV-cache snapshot | Model-specific, opaque, gigabytes per session |

SSP is none of these. It defines session state formally as the
**typed semantic diff against model priors** — storing only what the
model *cannot* regenerate from its training distribution.

The formal definition:

```
SESSION_STATE = diff(schema_defaults, current_state)
              filtered by non-regenerability
```

The **non-regenerability criterion** is the inclusion test:

> A value belongs in session state **if and only if** the model cannot
> reconstruct it from its training distribution and the current task
> summary in one inferential move.

---

## The Four Phases

| Phase | What It Does |
| --- | --- |
| **Extract** | Identify non-regenerable content in the live session. Construct a typed artifact. |
| **Consolidate** | Reorganize the artifact under the loop-closure test. Closed loops compress to topology. Dead branches exhale. The graph rebases. |
| **Persist** | Write the consolidated artifact to durable storage as a Session Record. |
| **Restore** | Reconstitute a live session from a stored artifact. Reinject trajectory and instruction nodes. Execute the Re-Entry Protocol. |

The **Consolidate** phase is the AI substrate projection of biological
sleep consolidation. It is the phase where session state is *improved*,
not just preserved.

---

## Fidelity Levels

| Level | Name | Fidelity | Available Today |
| --- | --- | --- | --- |
| 0 | No state | 0% | Default for most LLMs |
| 1 | Preference memory | 5–15% | Copilot Memory, ChatGPT Memory |
| 2 | Session summary | 25–40% | Manual / OpenAI Agents SDK |
| 3 | Structured State Envelope | 50–75% | **SSP spec** |
| 4 | Augmented Envelope + excerpts | 70–90% | **SSP spec (advanced)** |
| 5 | Native KV-Cache (SLS track) | ~100% | Requires infrastructure access |

Levels 3–4 are achievable today for any hosted model.
Level 5 is the theoretical ceiling — SLS defines it.

---

## The Procedural Layer

Three node types that no prior approach captures — canonical in SSP v2.0:

- **Instruction Nodes** — the procedural layer: what reasoning
  operations were running, what lookups were pending, what checks
  were flagged but not yet executed
- **Trajectory Nodes** — not just position but *direction*: where the
  session was going, what paths were explicitly declined
- **Regulatory State Nodes** — the user's cognitive state at session
  end: bandwidth, window width, and consolidation depth

Together these enable the **Re-Entry Protocol** — a structured
handshake that restores the *user*, not just the model, to the prior
session's reasoning thread.

---

## Repository Structure

```
/
├── README.md
├── specs/
│   ├── SSP_Specification.md           ← Canonical pipeline spec (v2.0)
│   ├── SLS_Initial_Proposal.md        ← KV-cache serialization (native track)
│   ├── SLS_Implementation_Guide.md    ← Engineering hooks + runtime
│   └── DSR_Specification.md           ← Semantic state envelopes (v0.2)
├── schemas/
│   └── sse.schema.json
├── examples/
    ├── example_sse_level3.json
    └── example_sse_level4.json

```

**Start with SSP_Specification.md.** It absorbs SLS and DSR as the two
fidelity tracks of the same pipeline. The SLS and DSR documents remain
as the underlying mechanism specifications.

---

## Status

| Document | Version | Status |
| --- | --- | --- |
| SSP Specification | v2.0 | Published |
| SLS Initial Proposal | v0.1 | Published — absorbed into SSP v2.0 |
| SLS Implementation Guide | v0.1 | Published — absorbed into SSP v2.0 |
| DSR Specification | v0.2 | Published — absorbed into SSP v2.0 |

---

## Independent Confirmations

Four 2026 systems, developed independently across three organizations,
converge on SSP's structural invariants:

- **Scroll** (Alibaba, arXiv:2608.21690) — environment-mediated context execution
- **Context Language Models** (UW/Allen AI, arXiv:2609.37725) — intrinsic model-controlled context rewriting
- **Engram, mHC, DSec, V4** (DeepSeek, 2026) — four architectural confirmations

Each arrived at one of SSP's invariants from a different engineering
direction, without knowledge of this specification. The convergence is
the evidence that the invariants are structural, not design choices.

---

## Related Work

SSP's Regulatory State Nodes and Re-Entry Protocol are grounded in a
broader mechanistic model of human cognitive architecture — specifically,
how prediction window width, autonomic state, and precision-gain interact
to determine working bandwidth. The framework composes across physiology,
cognition, language, AI, and social systems without structural failure.
Session state is the AI substrate projection of the same mechanism.

All this work is cosolidating towards my other repo-

**→ [The Manifold Schema](https://github.com/jtrthehax/manifold-schema)**

Manifold-schema covers this from physiological constraints, language, cognition.

---

## Citation

```
Robinson, J. (2026). SSP — Session State Pipeline: Extract, Consolidate, Persist, Restore. Zenodo. https://doi.org/10.5281/zenodo.20820098
```

---

## License

MIT License — Copyright (c) 2026 Joel Robinson
```
