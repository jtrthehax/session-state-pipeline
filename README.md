# **README.md (Draft v0.1)**

**LLM State Specification (SLS + DSR)**  
_A unified framework for latent‑state serialization, semantic state reconstruction, and multi‑session continuity in large language models._

---

## **Overview**

This repository contains the initial specification suite for **LLM statefulness**:

- **Serializable Latent State (SLS)** — a low‑level, infrastructure‑grade format for capturing and restoring the _actual latent state_ of a running LLM (KV‑cache, sampling state, system bindings).
- **Derived State Reconstruction (DSR)** — a high‑level, model‑agnostic method for reconstructing a functionally equivalent working state for _stateless hosted models_ where latent state access is impossible.

Together, SLS and DSR define the missing layer in the modern LLM stack:  
**a standardized, portable, inspectable representation of model state that enables session persistence, forking, migration, and multi‑session reasoning.**

This repo includes:

- **SLS Initial Proposal** — conceptual foundation and rationale
- **SLS Implementation Guide** — engineering details, hooks, and runtime considerations
- **DSR Specification** — semantic state envelopes for hosted models
- **Schemas and examples** for Structured State Envelopes (SSE)

---

## **Motivation**

LLMs today are effectively **stateless**. When a session ends or a context window fills, all accumulated understanding disappears:

- task structure
- decisions and rationale
- working hypotheses
- file and artifact state
- debugging context
- architectural constraints

Users must repeatedly re‑establish context, and systems cannot support long‑running or multi‑session workflows.

**SLS and DSR solve this.**

### **SLS** enables:

- deterministic replay
- session forking
- cross‑hardware migration
- auditability
- long‑running agents
- infrastructure‑level state management

### **DSR** enables:

- multi‑session continuity for hosted models
- model‑agnostic state transfer
- human‑inspectable working memory
- structured task and artifact tracking
- semantic reconstruction of working context

SLS is the ideal.  
DSR is the deploy‑today path.  
Together they form a complete statefulness architecture.

---

## **Repository Structure**

```
/
├── README.md
├── specs/
│   ├── SLS_Initial_Proposal.md
│   ├── SLS_Implementation_Guide.md
│   └── DSR_Specification.md
├── schemas/
│   └── sse.schema.json
├── examples/
│   ├── example_sse_level3.json
│   └── example_sse_level4.json
└── roadmap.md
```

---

## **Serializable Latent State (SLS)**

SLS defines a binary, portable, deterministic snapshot of an LLM’s latent state:

- KV‑cache tensors
- sampling RNG state
- model version + architecture hash
- system bindings
- metadata for compatibility and replay

SLS enables:

- **resume** — continue a session exactly where it left off
- **fork** — branch a session into multiple futures
- **migrate** — move a session across machines or runtimes
- **audit** — reproduce outputs bit‑for‑bit
- **checkpoint** — long‑running workflows without replay

SLS requires inference‑stack cooperation and is intended for self‑hosted or provider‑integrated environments.

---

## **Derived State Reconstruction (DSR)**

DSR is the semantic counterpart to SLS.  
It provides **80–90% of the value of latent‑state persistence** using only application‑layer engineering.

DSR defines the **Structured State Envelope (SSE)** — a typed JSON document containing:

- **Task Graph** — hierarchical task structure and progress
- **Artifact Registry** — files, code, schemas, and their states
- **Decision Log** — key decisions and rationale
- **Active Context** — current problem, hypotheses, constraints
- **Working Model (optional)** — critical verbatim excerpts

DSR enables:

- multi‑session continuity
- cross‑model portability
- human‑editable state
- resumable workflows
- agent memory without hallucinated summaries

DSR works with _any_ hosted model (OpenAI, Anthropic, Copilot, etc.) because it requires no access to internal model state.

---

## **Fidelity Levels**

The spec defines a 0–5 fidelity scale:

- **0** — no state
- **1** — preference memory
- **2** — narrative session summary
- **3** — structured state envelope
- **4** — structured envelope + critical excerpts
- **5** — native latent state (SLS)

Levels 3–4 are achievable today for all hosted models.  
Level 5 is ideal but requires provider support.

---

## **Status**

This is **v0.1**, an initial public release of the specifications.  
Future versions will refine:

- schema definitions
- extraction prompts
- reconstruction protocols
- compatibility layers
- reference implementations

Contributions, discussion, and feedback are welcome.

---

## **Citation**

A Zenodo DOI will be added after the first tagged release.

---

## **License**

	MIT License
	
	Copyright (c) 2026 Joel Robinson
	
	Permission is hereby granted, free of charge, to any person obtaining a copy
	of this software and associated documentation files (the "Software"), to deal
	in the Software without restriction, including without limitation the rights
	to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
	copies of the Software, and to permit persons to whom the Software is
	furnished to do so, subject to the following conditions:
	
	The above copyright notice and this permission notice shall be included in all
	copies or substantial portions of the Software.
	
	THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
	IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
	FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
	AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
	LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
	OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
	SOFTWARE.

