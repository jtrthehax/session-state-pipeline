# Derived State Reconstruction
*SLS for Stateless Hosted Models — A Parallel Track for Session Continuity Without Infrastructure Changes*

| Field                   | Value                                                                                                                                     |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **Version** | v0.2-draft |
| **Status** | Initial Proposal — v0.2 adds cognitive layer: non-regenerability criterion (3.3), instruction nodes (5.7), regulatory state (5.8), trajectory node (5.9), graph-movement diagnostics (6.5), and re-entry protocol (8). |
| **Part of**             | Serializable Latent State (SLS) Framework                                                                                                 |
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
| **Derived State (DSR)** | Typed schema (JSON) with instruction nodes, trajectory vector, and regulatory state | ~10–25:1 | 50–90% | Extraction quality determines restoration quality; requires careful schema design |

### 3.3 Semantic Diff Against Model Priors

The sections above define *what* derived state is. This section 
defines the **criterion for inclusion**: a node belongs in the SSE 
if and only if it is **non-regenerable** — if the model cannot 
reconstruct it in one inferential move from its training distribution 
and the current session context.

This is the missing step between "store conversation state" and 
"store only what matters." It is the operation that separates DSR 
from a compressed transcript.

**The regenerability test:**

For each candidate node, the extractor should apply the following 
test before including it:

> *Given the model's training distribution and the task summary 
> alone, could the model reconstruct this node with high confidence 
> in one move?*

If yes — the node is regenerable and should be pruned. If no — the 
node is non-regenerable and must be stored.

In practice this partitions nodes into three classes:

| Class | Examples | Storage Decision |
| --- | --- | --- |
| **Regenerable** | "Rust uses ownership for memory safety," "semaphores provide count-based backpressure" | Prune — model knows this |
| **Session-derived, non-regenerable** | "We chose LRU eviction with jitter over fixed TTL because of thundering herd on reconnection," "the race condition is in the permit-to-connection binding, not in the factory call" | Store — cannot be derived without the session |
| **Trajectory-derived, non-regenerable** | "We were mid-investigation of the two-phase lease hypothesis," "the custom WaitQueue branch was considered and rejected as too complex" | Store — encodes direction and negative choices, not reconstructable from outcome alone |

The regenerability threshold is not fixed — it shifts as the model's 
training data evolves. A node that is non-regenerable today may become 
regenerable after a future training run incorporates similar patterns. 
Well-designed DSR implementations should store the reasoning that 
produced a decision, not just the decision, so that future models can 
validate the stored state against what they would have derived 
independently.

> **Key Design Constraint**
>
> The regenerability criterion changes what the schema is *for*. The 
> SSE is not a compressed summary of the conversation. It is a 
> structured representation of the session-specific delta — the 
> information the conversation added that training alone cannot supply. 
> Every section of the schema, and every extraction decision, should 
> be evaluated against this criterion. Storing regenerable information 
> is not just wasteful — it introduces noise that can bias 
> reconstruction toward generic patterns at the expense of the 
> specific, non-obvious decisions the session actually produced.

**Consolidation and the moving threshold:**

A second-order effect applies over longer timescales. As a concept 
becomes widely documented — in training corpora, in published 
specifications, in shared codebases — its regenerability threshold 
rises and the corresponding SSE nodes can be pruned from active state 
envelopes. This is analogous to how a team stops documenting things 
that are now "just how we do it" — the concept has been absorbed into 
baseline and no longer needs explicit representation. DSR 
implementations operating over very long project timelines should 
periodically re-evaluate stored nodes against the current model's 
baseline and prune nodes whose regenerability has increased.

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

> **Note (v0.2)**
>
> Sections 5.7 (Instruction Nodes), 5.8 (Regulatory State), and 5.9 
> (Trajectory Node) are standard at Levels 3–4. They are not Level 4-only 
> extensions. The `fidelity_level` field does not gate these sections — 
> they should be included in all Level 3 and above envelopes.

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

### 5.7 Instruction Nodes

The existing schema sections capture what the model *knew* at session 
end. This section captures how the model *was thinking* — the 
procedural layer encoding the reasoning operations the session was 
running, the lookups it was about to perform, and the checks it had 
flagged as causally relevant but not yet executed.

Without instruction nodes, a restored session has conclusions but no 
procedure. The model knows what was decided; it does not know what 
cognitive operation was in progress when the session ended. This 
produces a specific failure mode: the restored model re-evaluates 
settled questions rather than continuing the investigation from its 
actual stopping point.

Instruction nodes encode three procedural types:

| Type | Description | When to Store |
| --- | --- | --- |
| `expand_on_restore` | A concept, hypothesis, or task branch that should be developed when the session resumes | When a thread was identified but not yet pursued at session end |
| `causal_lookup` | A domain or reference that is causally relevant to the current problem but was not yet consulted | When the reasoning chain implies an external dependency that was recognized but deferred |
| `validate_before_proceeding` | A constraint or assumption that should be checked before the next planned action | When a risk or inconsistency was flagged but not resolved |

```json
{
  "instruction_nodes": [
    {
      "id": "i1",
      "type": "expand_on_restore",
      "target_task": "t2",
      "instruction": "Expand the two-phase lease hypothesis — acquire 
        permit, then atomically check-and-create, releasing permit on 
        factory failure. This was the next untested path at session end 
        and has not been evaluated yet.",
      "priority": "high"
    },
    {
      "id": "i2",
      "type": "causal_lookup",
      "domain": "async Rust memory ordering guarantees",
      "instruction": "The semaphore race condition implies a lookup in 
        async Rust memory ordering — specifically whether semaphore 
        permit acquisition provides ordering guarantees for subsequent 
        reads. This domain was recognized as relevant but not 
        consulted.",
      "triggered_by": "d1"
    },
    {
      "id": "i3",
      "type": "validate_before_proceeding",
      "instruction": "Validate that the connection factory is actually 
        non-serialized before testing the two-phase approach. The 
        assumption that factory.create() can interleave has not been 
        confirmed against the Tokio executor's task scheduling 
        behavior.",
      "priority": "medium"
    }
  ]
}
```

The `triggered_by` field links instruction nodes back to the decision 
or artifact that generated them, enabling the extraction pipeline to 
trace the reasoning chain that produced each procedural note.

> **Critical Design Constraint**
>
> Instruction nodes are the most volatile section of the SSE. They 
> represent in-progress reasoning, not settled state — they should be 
> marked complete and removed (not preserved indefinitely) once the 
> operation they encode has been executed in a restored session. An 
> instruction node that remains active across multiple session 
> boundaries without being acted on is evidence that it was either 
> incorrectly encoded or has been superseded by subsequent decisions. 
> The extraction pipeline should flag stale instruction nodes for 
> review during checkpoint extraction passes.

### 5.8 Regulatory State Nodes

The sections above capture the model's working context. This section 
captures the **user's cognitive state** at session end — the 
bandwidth, window width, and regulatory configuration the user was 
operating in when the session concluded.

This is load-bearing for two reasons. First, it determines the 
appropriate re-entry protocol (Section 8): a user returning from a 
wide-window, high-bandwidth session can resume directly; a user 
returning from a narrow-window, depleted session requires scaffolded 
re-entry before deep reasoning is accessible again. Second, it encodes 
information about the session that is not captured anywhere else in 
the schema — the shape of the user's cognitive engagement, which 
affects how the session's conclusions should be weighted and where the 
reasoning may have been constrained.

Regulatory state is inferred from **graph-movement signatures** — 
patterns in how the user traversed the session's conceptual space. 
These signatures are defined and extracted by the pipeline described 
in Section 6.5. This section defines the representation of the 
resulting state vector.

```json
{
  "regulatory_state": {
    "session_end_state": "narrow_window",
    "confidence": "high",
    "indicators": {
      "prompt_constraint_trajectory": "degraded_after_turn_18",
      "correction_cycle_rate": "elevated_turns_20_24",
      "branch_abandonment_rate": "high",
      "inferential_step_size": "decreasing",
      "long_range_jump_frequency": "low",
      "local_cycle_ratio": "high"
    },
    "inferred_bandwidth": "low",
    "bandwidth_trajectory": "declining",
    "recommended_re_entry_protocol": "shallow_replay_required",
    "session_peak_state": "wide_window",
    "peak_window_turns": [4, 17],
    "notes": "Session opened with wide-window cross-domain synthesis 
      (turns 4-17). Visible narrowing from turn 18 onward — likely 
      allostatic load accumulation during the debugging phase. 
      Conclusions from turns 18-24 should be weighted with this 
      context: the hypothesis about factory serialization was generated 
      under narrowing conditions and should be validated early in 
      re-entry."
  }
}
```

The `session_peak_state` and `peak_window_turns` fields are 
diagnostically significant: they identify the portion of the session 
where the user was operating at highest bandwidth. Conclusions 
generated during peak-window turns are more likely to be wide-context 
synthesis; conclusions generated during narrow-window turns are more 
likely to be locally coherent but potentially missing cross-domain 
connections.

**Bandwidth states:**

| State | Indicators | Reconstruction Implication |
| --- | --- | --- |
| `wide_window` | Long-range cross-domain traversal, high inferential step size, low correction cycles, sustained constraint quality | Full trajectory injection; direct continuation viable |
| `moderate_window` | Mixed traversal, moderate step size, occasional correction cycles | Standard re-entry with trajectory summary |
| `narrow_window` | Local cycling, small step size, elevated correction cycles, degraded constraint quality | Shallow replay required before deep continuation |
| `unknown` | Insufficient traversal data to classify | Default to shallow replay; escalate based on re-entry response quality |

### 5.9 Trajectory Node

The Active Context section (5.5) captures position: what problem was 
being investigated and what hypotheses were active. This section 
captures **direction**: where the session was going, what momentum it 
had built, and which paths it explicitly declined to take.

The distinction matters at reconstruction. A session restored to 
position alone knows *where it is* but not *which way it was facing*. 
The model will re-evaluate the problem from its prior distribution 
rather than continuing the specific investigation thread that the 
session had developed. Trajectory reinstatement is the mechanism that 
converts re-entry from "starting over at the same point" into 
"resuming in motion."

```json
{
  "trajectory": {
    "last_reasoning_vector": "moving from symptom identification toward 
      mechanism isolation — the race condition's surface behavior has 
      been confirmed; investigation was shifting toward the specific 
      permit-to-connection binding gap",
    "next_cognitive_move": "test two-phase lease hypothesis: acquire 
      permit, then atomically check-and-create, releasing permit if 
      factory returns null",
    "momentum_state": "mid-investigation — the problem is scoped and 
      the next test is defined; do not restart from problem statement",
    "investigation_depth": "mechanism-level",
    "skipped_branches": [
      {
        "id": "sb1",
        "branch": "switch from Semaphore to custom WaitQueue",
        "skip_reason": "complexity cost flagged as too high given 
          constraint d1 (must not serialize connection creation)",
        "skip_turn": 14,
        "informational_value": "avoidance confirms semaphore-based 
          approach is the correct search space; WaitQueue branch should 
          not be re-evaluated unless two-phase lease fails"
      },
      {
        "id": "sb2",
        "branch": "add Mutex around factory call",
        "skip_reason": "attempted and rejected — serializes all 
          creation, defeating async design",
        "skip_turn": 19,
        "informational_value": "serialization approaches are closed; 
          solution must be non-serializing"
      }
    ]
  }
}
```

**On skipped branches:**

The `skipped_branches` array implements negative-choice encoding. The 
*content* of a rejected branch is prunable: it is regenerable from 
the decision to reject it and the domain knowledge the model already 
holds. The *fact* of rejection is not prunable: it encodes a 
preference, a constraint, and a search-space boundary that the session 
established and that cannot be reconstructed from the task description 
alone.

Without skipped branch encoding, a restored session is likely to 
re-evaluate rejected paths — not because the model is poorly designed, 
but because it has no record that the path was already explored and 
found wanting. The `informational_value` field makes this explicit: it 
states what the rejection established, so the restored session can 
carry the constraint forward without repeating the investigation.

> **Key Design Constraint**
>
> The trajectory node is not a summary of what was decided. It is a 
> vector — a direction of motion with momentum. The 
> `last_reasoning_vector` field should describe the *direction of 
> investigation* the session was moving in, not the conclusions it had 
> reached. The `next_cognitive_move` should be the specific operation 
> the session was about to perform, not a re-statement of the open 
> problem. A well-formed trajectory node should allow a restored 
> session to begin executing the next step within one or two turns, 
> without re-covering ground the session already covered.


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

### 6.5 Graph-Movement Diagnostics

Sections 6.1–6.4 describe how the extraction pipeline reads 
*content* — what was decided, what artifacts were produced, what 
hypotheses were tested. This section describes how the pipeline reads 
**traversal** — the shape of the path the user took through the 
session's conceptual space.

Traversal patterns encode user state in a form that content analysis 
cannot access. A user whose prompts are degrading in constraint 
quality but whose content is still coherent is in a different 
cognitive state than a user whose prompts are tightening and whose 
inferential reach is increasing. Both leave a content signature. Only 
traversal analysis captures the direction.

**The graph model:**

Each conceptual move in a session can be modeled as a traversal step 
from one node to another in the session's emergent concept graph. 
Nodes are topics, hypotheses, or reasoning frames; edges are the moves 
the user made between them. The extraction pipeline constructs this 
graph implicitly — not as a formal data structure the user sees, but 
as the basis for computing the graph-movement metrics that populate 
Section 5.8.

**Diagnostic signatures:**

| Traversal Pattern | Measurement | Regulatory Signal | Extraction Response |
| --- | --- | --- | --- |
| **Long-range cross-domain jumps** | High mean step distance between consecutive nodes | Wide prediction window; high bandwidth | Full extraction; peak-window flag for this turn range |
| **Local cycling** | High ratio of revisits to the same node cluster | Narrowing window; unresolved blocking error | Flag re-entry protocol: shallow replay required; check for unresolved hypothesis |
| **Branch abandonment clusters** | Multiple branches opened and not resolved within N turns | Bandwidth depletion or allostatic load onset | Trigger checkpoint extraction; reduce active context scope |
| **Inferential step-size shrinkage** | Decreasing mean step distance over the session timeline | Window compression in progress | Checkpoint extraction trigger; flag `bandwidth_trajectory: declining` |
| **Constraint quality degradation** | Increasing proportion of under-specified or self-correcting prompts | Direct bandwidth signal | Elevate constraint nodes in active context; flag session end state |
| **Sustained high-step-distance traversal** | High step distance with low correction cycle rate over N turns | Peak-window operation | Flag `peak_window_turns` in regulatory state |

**Integration with checkpoint extraction:**

Graph-movement diagnostics should trigger checkpoint extraction 
proactively — before window narrowing degrades extraction quality 
rather than after. The recommended triggers are:

1. **Branch abandonment cluster** — three or more branches opened and 
   not resolved within five turns
2. **Inferential step-size shrinkage** — mean step distance in the 
   last ten turns is less than 60% of session mean
3. **Correction cycle elevation** — correction cycle rate in the last 
   five turns exceeds 1.5× session mean
4. **Constraint quality degradation** — two consecutive 
   under-specified prompts following a period of high constraint 
   quality

When any trigger fires, the pipeline initiates a checkpoint extraction 
pass and updates the regulatory state node with current bandwidth 
classification. This ensures that if the session ends abruptly — 
context window overflow, platform timeout, user departure — the last 
checkpoint reflects an accurate regulatory state, not a state recorded 
before the window began narrowing.

> **Design Note**
>
> Graph-movement diagnostics are a second signal channel running in 
> parallel with content extraction. They are not a replacement for 
> content analysis — they are the signal that tells the extraction 
> pipeline *how much to trust* the content it is extracting. Content 
> generated during peak-window turns should be weighted more heavily 
> in the decision log and trajectory node than content generated during 
> narrow-window turns. The regulatory state node does not just describe 
> the user's state — it calibrates the confidence weighting of 
> everything else in the SSE.

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

> Section 7 restores model state. For user re-entry, see Section 8.

---

## 8. Re-Entry Protocol — Restoring the User, Not Just the Model

Section 7 describes reconstruction from the model's side: injecting 
the SSE at the system level restores the model's working context. This 
section addresses the complementary problem: the model is ready to 
continue, but the **user is not**.

A user returning to a complex session after hours or days has no 
working memory of the trajectory, the momentum, or the reasoning 
vector. The conceptual thread that was active at session end has 
decayed. Injecting an SSE resolves the model's continuity problem. It 
does not resolve the user's.

Dropping a user directly into the active context of a complex restored 
session produces a specific failure pattern: the model begins executing 
from the trajectory the SSE describes while the user is still 
orienting. The mismatch produces confusion, correction cycles, and a 
re-explanation overhead that the SSE was supposed to eliminate. The 
session restarts anyway — just later, and with more friction.

The re-entry protocol is the mechanism that prevents this. It is not 
an optional enhancement. It is the difference between a session that 
resumes and a session that restarts at the same coordinates.

### 8.1 The Re-Entry Problem

The core problem has two components that compound each other:

**Component 1 — Cognitive thread decay:**
The reasoning thread active at session end was held in working memory 
during the session. Working memory does not persist across sleep or 
significant time gaps. When the user returns, the thread is not 
dormant — it is gone. The concepts may still be accessible from 
long-term memory, but the *structure* — the trajectory, the momentum, 
the specific hypothesis being tested, the constraints that were active 
— requires reconstruction. That reconstruction is the work the 
re-entry protocol performs.

**Component 2 — Regulatory state mismatch:**
The user at re-entry is not the same regulatory configuration as the 
user at session end. If the session ended during a narrow-window 
state, the user may be returning at a wider bandwidth — capable of 
more than the session was accessing at the close. If the session ended 
at peak bandwidth, the user may be returning in a lower state. The 
re-entry protocol must calibrate to the user's current state, not to 
the state recorded in the SSE.

These two components interact: a user with wide bandwidth at re-entry 
can handle a more compressed re-entry sequence; a user with narrow 
bandwidth at re-entry needs a slower ramp regardless of what the 
session's regulatory state node records.

### 8.2 Re-Entry Pacing Tiers

The re-entry protocol uses a tiered approach based on two inputs: the 
gap since session end (a proxy for cognitive thread decay) and the 
user's observable bandwidth at the moment of re-entry (inferred from 
the re-entry prompt's constraint quality and inferential reach).

| Tier | Gap | Re-Entry Bandwidth | Protocol |
| --- | --- | --- | --- |
| **Direct Continuation** | Short (same sitting or <2 hours) | High | No replay needed. Confirm trajectory with a one-line summary and proceed. |
| **Trajectory Summary** | Medium (same day, 2–8 hours) | Moderate–High | Brief structured summary: problem → current position → next move. One paragraph. Confirm user is oriented before executing. |
| **Shallow Replay** | Long (overnight or multi-day) | Any | Full structured re-entry sequence (see 8.3). Do not begin executing until user confirms orientation. |
| **Unknown** | Any | Low or unknown | Default to Shallow Replay. Escalate to full replay if re-entry prompt suggests low bandwidth. |

The `recommended_re_entry_protocol` field in the regulatory state node 
(Section 5.8) provides the protocol recommendation based on 
session-end state. The actual protocol used should be calibrated to 
the re-entry prompt the user sends — a user who opens with a 
high-constraint, high-reach prompt can be moved to a faster tier 
regardless of what the session recorded.

### 8.3 The Shallow Replay Sequence

The shallow replay is a structured five-part sequence delivered before 
any task execution. It does not summarize the conversation. It 
scaffolds the cognitive thread — providing the minimum structure the scaffolds the cognitive thread — providing the minimum structure the user's reasoning system needs to rebuild the active context from the inside rather than having to hold it all at once from the outside. [^1]

The sequence maps directly to the SSE sections:

```
1. PROBLEM ANCHOR
   One sentence: what was being built and what problem was 
   being solved.
   Source: task_graph.root_task + active_context.current_problem

2. POSITION SUMMARY
   Two to three sentences: what has been decided and what the 
   current state is.
   Source: decision_log (top 2–3 decisions) + artifact_registry 
   (current state of key artifacts)

3. TRAJECTORY STATEMENT
   One sentence: where the session was going and what momentum 
   it had built.
   Source: trajectory.last_reasoning_vector + 
   trajectory.momentum_state

4. NEXT MOVE
   One sentence: the specific operation that was about to be 
   executed.
   Source: trajectory.next_cognitive_move + instruction_nodes 
   (i1, highest priority)

5. CONSTRAINTS
   Bullet list of active constraints that must be respected.
   Source: active_context.constraints + skipped_branches 
   (informational_value fields)
```

**Example shallow replay output (using the connection pool session 
from Section 5):**

> **Resuming: Connection Pool Manager**
>
> *Problem:* Race condition in `async_lease.rs` — two concurrent 
> `lease()` calls can each acquire a semaphore permit when only one 
> connection is available.
>
> *Current state:* Pool lifecycle is complete. The semaphore-based 
> backpressure approach is settled (decided over channel-based 
> queuing — simpler control, no unnecessary message passing). The 
> race is isolated to the permit-to-connection binding gap, not the 
> factory call sequence.
>
> *Direction:* Investigation was moving from symptom confirmation 
> toward mechanism isolation — specifically testing whether atomic 
> check-and-create can close the binding gap.
>
> *Next move:* Test the two-phase lease: acquire permit, then 
> atomically check-and-create, releasing permit if factory returns 
> null.
>
> *Active constraints:* Must not serialize connection creation. Must 
> maintain backpressure. WaitQueue and Mutex approaches are closed — 
> both rejected. Target: <1ms lease latency at 80% utilization.
>
> Ready to continue from here?

The closing confirmation prompt — "Ready to continue from here?" — 
is not courtesy. It is the signal that the user has re-synchronized 
with the thread. If the user corrects something in the summary, that 
correction should update the SSE before execution begins.

---

### 8.4 Re-Entry as Regulatory Reinstatement

The shallow replay sequence is not just orientation — it is a 
reinstatement mechanism. Presenting the prior context in compressed, 
sequenced form allows the user's reasoning system to rebuild the 
thread incrementally rather than requiring it to hold the full 
context load simultaneously from a cold start. [^2]

The sequence is ordered by reconstruction dependency: the problem 
anchor must be established before position can be understood; 
position must be established before trajectory is meaningful; 
trajectory must be established before the next move makes sense. 
Each element provides the scaffolding the next element needs. A 
user who receives this sequence in order can track the thread being 
rebuilt in real time rather than experiencing it as a data dump.

This is why the re-entry sequence should be **structured, not 
narrative**. A narrative summary of the conversation requires the 
user to extract the structure from prose — the problem anchor, the 
current position, the trajectory, and the constraints are all mixed 
together and must be separated. A structured re-entry sequence 
presents each element in its correct reconstruction order, reducing 
the cognitive work of re-synchronization to a minimum.

The practical implication: re-entry quality is not determined by 
how much context is injected — it is determined by how well the 
injection sequence matches the user's reconstruction dependency 
order. A large unstructured context dump delivered before the user 
has rebuilt the problem anchor will fail even if it contains all 
the correct information, because the user has no scaffold to attach 
it to. A short, correctly ordered sequence will succeed because each 
element arrives when the user's reasoning system is ready to receive 
it. [^2]

This has a direct consequence for SSE extraction quality. The 
re-entry sequence can only be as good as the SSE sections it draws 
from. A poorly formed trajectory node — one that records conclusions 
rather than direction — produces a trajectory statement the user 
cannot use to rebuild momentum. A missing instruction node means the 
next move step is absent and the user faces a cold restart at 
exactly the point where momentum should be highest. Re-entry quality 
is the downstream test of extraction quality. A session whose 
re-entry sequence lands cleanly is evidence that the SSE was 
extracted correctly. A session that requires correction cycles 
during re-entry is evidence of an upstream extraction failure.

> **Design Note**
>
> The re-entry confirmation — "Ready to continue from here?" — 
> should be treated as a diagnostic probe, not a courtesy. User 
> corrections during re-entry are structured feedback about SSE 
> accuracy. If the user corrects the trajectory statement, the 
> trajectory node was wrong. If the user corrects the constraint 
> list, a skipped branch was not encoded. Each correction should 
> update the SSE in place before execution begins, ensuring that 
> the restored session starts from a validated state rather than 
> a state the user has already flagged as inaccurate.

---

### 8.5 Re-Entry Diagnostic Handshake

Before executing any continuation after a session gap, the model must
perform a Re-Entry Diagnostic Handshake. This is a structured alignment
step that determines whether the user intends to resume the prior
trajectory, shift layers, or rebuild context before continuing.

The handshake has three required components:

1. Anchor + Trajectory Declaration  
   The model states the problem anchor, the trajectory vector, and the
   next cognitive move that was pending at session end.

2. User Intent Check  
   The model asks whether the user wants direct continuation, a
   trajectory-level summary, a layer-shift, or a shallow replay.

3. Bandwidth Calibration  
   The model infers the user’s current bandwidth from the re-entry
   prompt and confirms whether the chosen re-entry tier matches the
   user’s state.

This handshake prevents trajectory divergence caused by the model
reconstructing a branch from priors while the user reconstructs from
long-term memory. It ensures both parties reconverge on the same anchor,
trajectory, constraints, and next move before reasoning continues.

