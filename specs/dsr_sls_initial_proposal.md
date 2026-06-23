# Derived State Reconstruction
*SLS for Stateless Hosted Models — A Parallel Track for Session Continuity Without Infrastructure Changes*

| Field | Value |
| --- | --- |
| **Version** | v0.1-draft |
| **Date** | April 2026 |
| **Status** | Initial Proposal |
| **Part of** | Serializable Latent State (SLS) Framework |
| **Companion Documents** | "Serializable Latent State — Initial Proposal for LLM Save Files," "SLS Implementation Guide," "Hosted SLS — API-Level State Persistence" |

---

## 1. Abstract

Derived State Reconstruction (DSR) extends the Serializable Latent State framework to stateless hosted models — systems like Microsoft Copilot, ChatGPT, and Claude where direct access to the inference stack's KV-cache is impossible. DSR defines a structured, serializable representation of conversational working state that can be extracted during a session, persisted to storage, and injected into a new session to reconstruct equivalent (not identical) working context.

DSR is not a degraded fallback from native KV-cache serialization. It is a **parallel track** — one that ships to the most users fastest because it requires zero inference-infrastructure changes. It is pure application-layer engineering. The core thesis is straightforward: you do not need the literal KV-cache to get 80%+ of the value. A structured "Derived State" representation, extracted from the conversation and injected at session start, can restore a model's working context well enough to resume complex multi-session tasks — writing a complicated script, managing a multi-file project, iterating on a systems architecture — without forcing the user to re-explain everything from scratch.

This is the **highest-leverage track** in the SLS proposal. It benefits every user of every hosted model. It requires no provider cooperation. It can be built and shipped today.

---

## 2. Motivation — The Session Boundary Problem

Users of hosted models hit a hard wall when a session ends or a context window fills. All accumulated understanding evaporates: the file structure being discussed, the architectural decisions made, the debugging hypotheses tested, the stylistic preferences demonstrated, the constraints agreed upon. The model returns to a blank slate. The user must spend significant effort re-establishing context in the new session — re-uploading files, re-explaining decisions, re-stating constraints, and hoping the model infers the same working state it had before.

This is especially painful for complex, multi-session tasks:

- **Iterating on a complicated script over multiple sessions** — the model forgets the module structure, the design decisions, and the specific bug being investigated
- **Building and refining a systems architecture** — each session loses the accumulated understanding of component interactions, trade-offs evaluated, and constraints discovered
- **Managing a multi-file project with interdependencies** — the model cannot recall which files exist, their relationships, or their current state
- **Long-running research synthesis** — source evaluations, thematic threads, and analytical frameworks dissolve between sessions

### 2.1 Why Naive Full-History Replay Fails

The obvious solution — replaying the entire conversation history into the new session — does not work for four compounding reasons:

| Failure Mode | Mechanism | Consequence |
| --- | --- | --- |
| **Finite context windows** | Context windows are bounded (128K–1M tokens today). Multi-session work easily exceeds this. | History physically cannot fit. Older context is truncated, losing early decisions and context. |
| **Lost in the Middle** | Transformer attention disproportionately weights the beginning and end of context, degrading retrieval for information in the middle. | Even when history fits, critical mid-conversation decisions are effectively invisible to the model. Signal is diluted by noise. |
| **Quadratic attention cost** | Self-attention scales O(n²) with sequence length. Replaying 100K tokens of history is computationally expensive. | Slow inference, high cost per query, poor user experience — especially for hosted models billing by token. |
| **Context poisoning** | Stale context — abandoned hypotheses, superseded decisions, incorrect intermediate outputs — remains in the history alongside current state. | The model may resurrect dead-end approaches, apply outdated constraints, or contradict decisions that were made later in the conversation. |

Full-history replay treats all tokens as equally important. They are not. The information-theoretic content of a 50,000-token conversation can typically be captured in 2,000–5,000 tokens of structured state — a 10–25x compression ratio that preserves what matters and discards what doesn't.

---

## 3. Concept — What Is Derived State?

**Derived State** is a structured, human-readable, machine-parseable representation of the working context accumulated during one or more LLM sessions. It captures *what* the model was working on, *what* decisions were made, *what* artifacts exist, and *what* working hypotheses or constraints are active — without capturing the literal attention patterns or KV-cache tensors.

It is "derived" in the precise sense: it is extracted from conversational content rather than dumped from internal model state. It is a semantic reconstruction, not a bitwise copy.

### 3.1 Key Properties

- **Lossy but sufficient.** It discards the vast majority of conversational tokens while preserving the information-theoretic core needed for task continuation. The conversation about *why* we chose LRU eviction is discarded; the *decision* that we chose LRU eviction (and the rationale) is preserved.
- **Structured, not narrative.** Unlike a conversation summary, it uses a typed schema with explicit sections for different kinds of state — task graph, artifact registry, decision log, active context. Each section serves a distinct function in reconstruction.
- **Model-agnostic.** The same derived state can be injected into different model versions or even different model families, because it is text-based. An SSE extracted from a Copilot session can be injected into Claude, and vice versa.
- **Human-inspectable.** Users can read, edit, and curate their state files. Unlike opaque KV-cache blobs, a state envelope is transparent — a user can see exactly what the model "remembers" and correct it if wrong.
- **Composable.** States from different sessions can be merged, differenced, or selectively applied. A user can fork a state, explore two directions, and merge the results.

### 3.2 Contrast with Existing Approaches

| Approach | Structure | Compression | Fidelity | Key Limitation |
| --- | --- | --- | --- | --- |
| **Conversation logs** | None — raw transcript | None (1:1) | Complete but noisy | Too large, no semantic extraction, context poisoning from stale content |
| **Chat summaries** | Narrative prose | ~50:1 | Unpredictable | Lossy in unpredictable ways; not machine-parseable; no typed schema |
| **KV-cache snapshots** | Tensor data | None (1:1) | ~100% | Model-specific, opaque, enormous (GBs), requires infrastructure access |
| **Memory / preferences** | Sparse key-value pairs | ~1000:1 | 5–15% | Captures "user likes X" but not "we were debugging a race condition on line 47" |
| **Derived State (DSR)** | Typed schema (JSON) | ~10–25:1 | 50–90% | Extraction quality determines restoration quality; requires careful schema design |

---

## 4. The Fidelity Spectrum

State reconstruction exists on a spectrum from zero context to perfect recall. Derived State occupies Levels 2–4 — the "sweet spot" that is achievable today with pure application-layer engineering and sufficient for the vast majority of task continuation scenarios.

| Level | Name | What's Captured | Compression | Fidelity | Who Has This |
| --- | --- | --- | --- | --- | --- |
| 0 | **No State** | Nothing — every session starts cold | N/A | 0% | Default for most LLMs |
| 1 | **Preference Memory** | User preferences, facts, contact info | ~1000:1 | 5–15% | Copilot Memory, ChatGPT Memory |
| 2 | **Session Summary** | Narrative summary of what happened, key decisions, outcomes | ~100:1 | 25–40% | OpenAI Agents SDK (manual) |
| 3 | **Structured State Envelope** | Typed schema: task graph, artifact registry, decision log, active constraints, working model | ~20:1 | 50–75% | **This proposal** |
| 4 | **Augmented State Envelope** | Level 3 + compressed verbatim excerpts of critical artifacts (code, schemas, key passages) | ~5:1 | 70–90% | **This proposal (advanced)** |
| 5 | **Native KV-Cache** | Literal attention state tensors | 1:1 | ~100% | SLS Core Spec (requires infrastructure access) |

> **Key Insight**
>
> Level 3–4 is the sweet spot. It is achievable with pure application-layer engineering. It is sufficient for task continuation. It is human-inspectable. It is model-agnostic. Level 5 is theoretically ideal but requires deep infrastructure access that hosted API consumers do not have — and may never get. DSR delivers 80%+ of the value of native SLS with 0% of the infrastructure dependency.

---

## 5. The State Envelope — Schema Design

The **Structured State Envelope (SSE)** is the core data structure of DSR. It is what gets serialized, persisted, transferred, and injected. It is a typed JSON document with six sections, each serving a distinct function in reconstruction.

### 5.1 Envelope Metadata

Top-level metadata for version tracking, session chaining, and provenance:

```json
{
  "envelope_version": "0.1",
  "session_id": "sess_20260410_142337_a7c3",
  "parent_session_id": "sess_20260409_091205_f1b2",
  "created_at": "2026-04-10T14:23:37Z",
  "model_id": "copilot-2026-04",
  "task_summary": "Build a connection pool manager for the microservice gateway",
  "fidelity_level": 3
}
```

The `parent_session_id` field enables session chaining — a linked list of sessions working on the same task. The `fidelity_level` field indicates whether the envelope includes verbatim excerpts (Level 4) or structure only (Level 3).

### 5.2 Task Graph

The hierarchical structure of what is being worked on. Each subtask has a status, description, and dependency links:

```json
{
  "task_graph": {
    "root_task": "Build a connection pool manager for the microservice gateway",
    "subtasks": [
      {
        "id": "t1",
        "description": "Design the pool lifecycle (create, lease, release, evict)",
        "status": "completed",
        "outcome": "Implemented with configurable min/max pool size, idle timeout, and health-check interval"
      },
      {
        "id": "t2",
        "description": "Implement async connection leasing with backpressure",
        "status": "in_progress",
        "current_focus": "Debugging race condition in lease() when pool is at capacity",
        "blocked_by": null
      },
      {
        "id": "t3",
        "description": "Add observability (metrics, logging, tracing)",
        "status": "not_started",
        "depends_on": ["t1", "t2"]
      }
    ]
  }
}
```

The task graph gives the model immediate orientation: what is done, what is in progress, what is blocked. Without this, the model must infer task structure from conversation fragments — an error-prone process that frequently produces inconsistent results.

### 5.3 Artifact Registry

What files and artifacts exist, their current state, and their key structures:

```json
{
  "artifacts": [
    {
      "id": "a1",
      "type": "source_file",
      "path": "src/pool/manager.rs",
      "description": "Main pool manager struct and lifecycle methods",
      "last_known_state": "~180 lines, compiles, passes 4/6 tests",
      "key_structures": ["ConnectionPool struct", "PoolConfig", "LeaseHandle"]
    },
    {
      "id": "a2",
      "type": "source_file",
      "path": "src/pool/async_lease.rs",
      "description": "Async lease/release implementation with semaphore-based backpressure",
      "last_known_state": "~120 lines, has a race condition in the semaphore acquire path",
      "key_structures": ["LeaseFuture", "BackpressurePolicy"]
    },
    {
      "id": "a3",
      "type": "test_file",
      "path": "tests/pool_integration.rs",
      "description": "Integration tests for pool lifecycle",
      "last_known_state": "6 tests, 4 passing, 2 failing (related to async lease race condition)"
    }
  ]
}
```

The artifact registry answers the question: "What files exist and what shape are they in?" This is the single most common source of re-explanation overhead in multi-session coding work.

### 5.4 Decision Log

Key decisions made and their rationale. This is critical for consistency across sessions — without a decision log, the model may re-evaluate settled questions and arrive at different conclusions:

```json
{
  "decisions": [
    {
      "id": "d1",
      "decision": "Use tokio::sync::Semaphore for backpressure instead of channel-based queuing",
      "rationale": "Semaphore gives direct count-based control; channel adds unnecessary message passing",
      "made_at": "session 1",
      "alternatives_considered": ["bounded channel", "custom atomic counter"]
    },
    {
      "id": "d2",
      "decision": "Pool eviction uses LRU with jitter, not fixed TTL",
      "rationale": "Fixed TTL causes thundering herd on reconnection; jitter distributes load",
      "made_at": "session 2"
    }
  ]
}
```

### 5.5 Active Context

The "working memory" — what the model should have top-of-mind when the session resumes. This is the most volatile section: it captures the immediate state of investigation, not settled decisions.

```json
{
  "active_context": {
    "current_problem": "Race condition in async_lease.rs: when the pool is at max_size and two tasks call lease() simultaneously, the semaphore acquire can succeed for both but only one connection is available",
    "hypotheses": [
      "The semaphore permit count is being decremented before the connection is actually created, creating a window where permits and connections are out of sync",
      "The connection factory's async creation isn't being serialized — two factory calls can interleave"
    ],
    "last_action": "Added a Mutex around the connection factory call, but this serializes all creation and defeats the purpose of async",
    "next_steps": [
      "Try a two-phase lease: acquire semaphore permit, THEN atomically check-and-create from pool, releasing permit on failure",
      "Consider switching from Semaphore to a custom WaitQueue that manages the permit-to-connection binding"
    ],
    "constraints": [
      "Must not serialize connection creation — the whole point is async concurrent leasing",
      "Must maintain backpressure — callers should await when pool is exhausted",
      "Target: <1ms lease latency at 80% pool utilization"
    ]
  }
}
```

### 5.6 Working Model (Level 4 Only)

Compressed verbatim excerpts of the most critical content. This section is only present in Level 4 envelopes and significantly increases fidelity at the cost of size:

```json
{
  "working_model": {
    "critical_excerpts": [
      {
        "artifact_id": "a2",
        "region": "lease() method, lines 34-67",
        "content": "pub async fn lease(&self) -> Result<LeaseHandle, PoolError> {\n  let permit = self.semaphore.acquire().await?;\n  // BUG: race condition window between permit acquisition\n  // and connection retrieval\n  let conn = self.pool.try_get().unwrap_or_else(|| {\n    self.factory.create().await // <-- not serialized\n  });\n  Ok(LeaseHandle::new(conn, permit))\n}",
        "annotation": "This is the buggy code — the race is between permit.acquire() succeeding and pool.try_get() returning None, triggering an unserialized factory.create()"
      }
    ]
  }
}
```

The working model transforms reconstruction from "the model knows *about* the code" to "the model can *see* the code." For debugging tasks, this difference is often decisive.

---

## 6. Extraction Pipeline — How State Gets Captured

Extraction is the process of transforming a raw conversation into a structured state envelope. It is the most quality-critical component of DSR: a bad extraction poisons all future sessions. Two extraction modes serve different needs.

### 6.1 Continuous Extraction (Background)

The application layer continuously maintains and updates the state envelope as the conversation progresses. After each assistant turn:

1. The LLM's response is analyzed for **state-changing events** — task status changes, new decisions, artifact modifications, hypothesis updates.
2. A lightweight extraction prompt (or rule-based parser) updates the relevant SSE sections. Only changed sections are rewritten.
3. The envelope is persisted after each turn. This is cheap — a typical SSE is 2–5 KB of JSON.

This is analogous to **autosave** in video games. The user never has to think about it. The state is always current. If the session crashes or the context window fills, the last-persisted SSE is available for reconstruction.

> **Advantage**
>
> Continuous extraction can be verified turn-by-turn. If the extractor misinterprets a decision in turn 12, it can be caught and corrected in turn 13. Errors do not compound silently.

### 6.2 Checkpoint Extraction (Explicit)

At defined points — end of session, user-triggered "save," before a risky operation — a dedicated extraction pass runs:

1. The full conversation history (or as much as fits in context) is scanned.
2. A specialized extraction model (or the same model with an extraction prompt) generates a complete SSE from the conversation.
3. The checkpoint SSE replaces the incremental one as the canonical state.

This is analogous to **manual save** — higher fidelity because it processes the full context holistically, resolving any inconsistencies that accumulated during continuous extraction.

### 6.3 Extraction Prompt Design

The extraction prompt is the interface between raw conversation and structured state. A well-designed extraction prompt is the single highest-leverage component in the DSR pipeline. Example template:

```
Given the following conversation between a user and an AI assistant, extract a
Structured State Envelope capturing:

1. TASK GRAPH: The hierarchical task structure with status (completed /
   in_progress / not_started), outcomes for completed tasks, current_focus for
   in-progress tasks, and dependency links.
2. ARTIFACT REGISTRY: All files, documents, or artifacts discussed, with their
   type, path, current state description, and key structures or components.
3. DECISION LOG: Key architectural or design decisions with rationale, when they
   were made, and what alternatives were considered and rejected.
4. ACTIVE CONTEXT: The current problem being investigated, working hypotheses,
   the last action taken, planned next steps, and active constraints that must
   be respected.
5. WORKING MODEL (if fidelity_level=4): Verbatim excerpts of critical code,
   schemas, or passages that are essential for the current investigation.

Output as JSON matching the SSE schema v0.1. Prioritize accuracy over
completeness — it is better to omit uncertain information than to include
incorrect state.
```

> **Critical Design Constraint**
>
> Extraction quality determines restoration quality. A single misattributed decision or incorrect artifact state in the SSE will propagate through all future sessions. The final line of the extraction prompt — "prioritize accuracy over completeness" — is deliberate. An incomplete-but-correct SSE is strictly better than a complete-but-wrong one.

### 6.4 Extraction Mode Comparison

| Dimension | Continuous (Background) | Checkpoint (Explicit) |
| --- | --- | --- |
| **Trigger** | After every assistant turn | End of session, user-triggered, pre-risky-operation |
| **Scope** | Incremental — only processes the latest turn | Holistic — processes the full conversation |
| **Latency** | Low — lightweight update, non-blocking | Higher — full extraction pass |
| **Consistency** | Can drift if incremental updates conflict | Internally consistent (single-pass) |
| **Crash safety** | Always have recent state | Only have state from last checkpoint |
| **Preferred for** | Complex, long-running sessions | Session boundaries, before major context shifts |

In practice, both modes should be used together. Continuous extraction maintains a running SSE; checkpoint extraction periodically reconciles and validates it.

---

## 7. Reconstruction Protocol — How State Gets Injected

Reconstruction is the reverse of extraction: a persisted SSE is injected into a new session to restore working context. The protocol has three phases.

### 7.1 System-Level Injection

The SSE is injected as structured context at the **system/instruction level** — NOT as simulated conversation history. This distinction is critical:

- **System-level context receives highest attention priority.** It does not suffer from "Lost in the Middle" degradation because it occupies the beginning of the context window.
- **It avoids the model trying to "continue" a conversation that did not happen in this session.** Injecting state as fake user/assistant turns creates confusion about what was said in this session vs. the previous one.
- **It frames the state as ground truth rather than something to be debated.** System-level context has implicit authority — the model treats it as established fact, not conversational claim.

### 7.2 Priming Sequence

After injection, a brief priming exchange establishes that the model has absorbed the state and is ready to continue:

```
[System prompt includes full SSE content]

User: "I'm continuing work on the connection pool manager for the microservice
gateway. We left off debugging the race condition in async_lease.rs — the
semaphore acquire succeeds for two concurrent callers but only one connection is
available. Let's try the two-phase lease approach."
```