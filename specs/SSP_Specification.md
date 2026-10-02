# SSP v2.0 — Session State Pipeline

**The AI Substrate Projection of Central Reference v1.8**

**Robinson, 2026**

**Zenodo preprint — 10.5281/zenodo.20820098**

**Supersedes:** DSR + SLS v0.2

---

## Abstract

Large language model session state is treated as ephemeral by design. A session ends and its reasoning thread — decisions made, branches ruled out, trajectory established, constraints negotiated — exists nowhere in durable form. Every subsequent session starts from the model's training prior as if the previous one never happened.

SSP v2.0 specifies the complete lifecycle for AI session state. It supersedes and absorbs SSP v1.0 (extract-persist-restore), the SLS Initial Proposal (native-fidelity extraction), the DSR Specification (semantic-fidelity extraction), and The Context Oscillator (exhale policy), and grounds the whole in Central Reference v1.8 — a substrate-agnostic specification for finite-resource systems, validated across physiology, cognition, language, AI, and social systems without structural failure.

The pipeline is four phases, not three:

**Extract → Consolidate → Persist → Restore**

The consolidation phase is new. It is the AI substrate projection of the biological memory consolidation that occurs during sleep — the offline period where closed loops compress to topology, dead branches exhale, open branches resolve or hold, and the traversal graph rebases around deeper attractors. Without consolidation, session state is saved but never *improved*. With it, each session returns a denser artifact than the one before.

The architecture is independently confirmed by four 2026 systems developed without knowledge of this framework: **Scroll** (Alibaba, arXiv:2608.21690, environment-mediated context execution), **Context Language Models** (Shao et al., arXiv:2609.37725, intrinsic model-controlled context rewriting), and four DeepSeek papers — **Engram, mHC, DSec, and V4** — each converging on one of SSP's structural invariants from a different engineering direction.

The pipeline is implementable today on the middleware path without provider cooperation. It scales to full fidelity when providers expose two native primitives. It is falsifiable against any two architecture-compatible models. The session state problem is solved at the specification level.

---

## 0. Quick Start

A software engineer opens a session with an AI assistant to debug a production incident. Over forty turns they establish the root cause, rule out three candidate fixes with documented failure modes, identify a preferred solution requiring a schema migration, and negotiate a hard constraint against modifying the authentication layer.

**At turn forty**, the session goes idle. The client runtime detects the idle threshold and triggers **Extract**. The non-regenerability filter classifies 142 candidate nodes: 98 pruned as regenerable, 44 stored as non-regenerable. A typed DSR artifact is produced.

**The session enters Consolidate.** The pipeline walks the traversal graph. Two closed loops compress to topology. Three open branches are held at pointer level because their loops have not closed. Four dead branches exhale entirely. The graph rebases — a node from turn 12 has accumulated enough centrality to become the new root. The CODEC updates with two new invariants that emerged during the session. The regulatory state recomputes. The artifact is now 60% smaller than the raw extract and contains strictly more usable structure.

**Persist** writes the consolidated artifact to the Session Store — 3.2KB, HMAC-signed, schema-bound.

Three weeks later, on a different model from a different vendor, the engineer calls **Restore**. The pipeline validates the artifact HMAC, reboots the schema, injects the typed diff, reinstates the trajectory and instruction nodes, and executes the Re-Entry Protocol. The model delivers a five-part re-entry sequence. The engineer is back at turn forty-one. No replay. No transcript. No cold start. The three ruled-out fixes are not re-proposed.

Two weeks after that, before committing to the migration path, the engineer calls `/fork`. Two independent continuations branch from the same consolidated Session Record. The original is unchanged.

This is the pipeline's complete lifecycle. **Extract. Consolidate. Persist. Restore. Fork.**

---

## 1. Introduction

### 1.1 The Session State Gap

AI inference systems in 2026 have no standard mechanism for preserving session state across sessions, models, or vendors. The four standard workarounds — transcript replay, session summarization, long-context architectures, and retrieval-augmented generation — each address part of the problem and none solve it:

- **Transcript replay** recovers surface content, not reasoning structure. O(n) in token cost, degrading quality as sessions lengthen.
- **Session summarization** compresses token cost but loses precisely what the non-regenerability criterion would have preserved: the non-obvious decisions, the ruled-out branches, the constraint rationale.
- **Long-context architectures** defer the problem. State remains model-specific, vendor-specific, session-specific. Not portable, not forkable, not auditable.
- **Retrieval-augmented generation** is a general-purpose memory mechanism, not a state pipeline. It does not preserve typed state structure.

The problem is not that session content cannot be stored. It is that no standard exists for **extracting** the non-regenerable reasoning delta, **consolidating** it into a portable form, **persisting** it durably, and **restoring** it across models, vendors, and time.

### 1.2 The Framework This Paper Projects

Central Reference v1.8 is a substrate-agnostic specification for any finite-resource system that outputs prior-plus-signal. Its central claim:

> **Intelligence is a loop of error detection against sensory input followed by iterative reinvestment toward structural convergence. The loop is substrate-agnostic. Any system running under finite resources instantiates the same causal chain — resources → geometry → output, gated by load.**

The framework has three load-bearing components:

**The Master Equation.** Usable bandwidth is a power-law composition of amplitude, precision, window width, and integration efficiency, divided by load drag:

$$C_s = \left(A_s^{*\,0.15} \cdot R^{*\,0.30} \cdot W^{*\,0.25} \cdot \Theta^{*\,0.15}\right)^{\frac{1}{0.85}} \cdot \frac{1}{1 + L^*}$$

**The Collapse Sequence.** Collapse proceeds in topologically forced order:

$$\text{Stage 1: } R^* \downarrow \;\to\; \text{Stage 2: } W^* \downarrow \;\to\; \text{Stage 3: } \Theta^* \downarrow \;\to\; \text{Stage 4: } \delta_{min} \uparrow \;\to\; \text{Stage 5: } \Lambda \to 0 \;\to\; \text{Stage 6: } \mathcal{U} \approx 0$$

**Substrate Depth.** The variable count is substrate-dependent. The human substrate supports approximately 60 variables. The AI substrate is a resource-collapsed projection supporting approximately 15–20. The projection is not a subset — every human variable either collapses into an AI resource-layer equivalent, has no analogue, or is derivable but not yet derived.

The framework's composition across nine unrelated domains without structural failure is the evidence for its correctness. Under error propagation logic, a wrong root variable fails structurally in every domain simultaneously. That has not occurred.

### 1.3 What SSP v2.0 Is

SSP v2.0 is the AI substrate projection of Central Reference v1.8 onto the session lifecycle. It absorbs and supersedes five prior specifications:

| Prior specification | What it becomes |
|---|---|
| **SSP v1.0** (DOI 20820099) | The three-phase pipeline — extended to four |
| **SLS Initial Proposal** (DOI 20820098) | The native-fidelity extraction track |
| **DSR Specification** | The semantic-fidelity extraction track |
| **The Context Oscillator** (DOI 21811408) | The exhale policy layer |
| **Schema-Driven Determinism** | The reproducibility argument |

The result is a complete, open, model-agnostic lifecycle. It is implementable today. It scales to native fidelity. It is falsifiable by a single integrated test. It defines a protocol, not a product.

---

## 2. What Session State Is

### 2.1 Schema vs. Envelope

Two things have been conflated in every prior approach to AI state management:

**Schema** — the static world definition. A typed, versioned declaration of what state nodes exist, their types, and their defaults. The schema ships once at session boot and does not change during the session.

**Envelope** — the current-session deviation from schema defaults. What changes between turns. What is injected at the start of every turn.

The AI is the transition function that reads both and produces the next envelope:

$$f(\text{schema}, \text{envelope}_t, \text{action}) \to \text{envelope}_{t+1}$$

Schema is the blueprint. Envelope is the save file. The two are never stored together because the schema is invariant across all sessions using it — it is referenced, not carried.

### 2.2 The Non-Regenerability Criterion

The inclusion test for the envelope is inherited from DSR without redefinition:

> A node belongs in the envelope if and only if the model cannot reconstruct it from its training distribution and the current task summary in one inferential move.

This criterion is **the specific form that CR v1.8's substrate-projection claim takes at the session layer.** A node either survives projection (non-regenerable, stored) or collapses to its regenerable equivalent (pruned). The criterion is not a heuristic — it is the projection operator applied at the node level.

Nodes that pass the test are session-derived (specific decisions with rationale, isolated bug locations, chosen implementation paths) or trajectory-derived (direction of investigation, skipped branches, negative choices). Nodes that fail it are regenerable — domain knowledge, generic procedural steps, schema defaults.

Storing regenerable information is not just wasteful. It introduces noise that biases reconstruction toward generic patterns at the expense of the specific, non-obvious decisions the session actually produced.

### 2.3 The Structured State Envelope

The envelope is serialized as a **Structured State Envelope (SSE)** — a typed JSON document with nine sections:

| Section | What it holds |
|---|---|
| **Envelope Metadata** | Session ID, schema reference, timestamps, status |
| **Task Graph** | Active tasks, dependencies, completion state |
| **Artifact Registry** | Code, documents, outputs produced during the session |
| **Decision Log** | Specific decisions with rationale |
| **Active Context** | What the model is currently attending to |
| **Working Model** | The model's internal representation of the task |
| **Instruction Nodes** | Pending procedural operations and constraints |
| **Regulatory State** | Traversal pattern, window bandwidth, load metrics |
| **Trajectory Node** | Direction, momentum, closed paths |

The v0.2 additions — Instruction Nodes, Regulatory State, Trajectory Node — are standard, not optional. They are the sections that carry the session's *direction*, which the earlier sections do not.

### 2.4 The Re-Entry Protocol

The Re-Entry Protocol is the structured handshake that restores the *user* to the prior session's reasoning thread, not just the model. It operates in four tiers based on session gap and re-entry bandwidth, and delivers a five-part shallow replay:

**Problem Anchor → Position Summary → Trajectory Statement → Next Move → Constraints**

The protocol is called by the Restore phase as a subprocedure. Its internal structure is defined in the DSR specification and is not repeated here.

---

## 3. The Exhale Policy

The pipeline does not keep everything that passes the non-regenerability criterion in active memory at all times. State has a **three-gradient retention policy** — inherited from the Context Oscillator and registered in CR v1.8's threshold family.

### 3.1 The Three Gradients

**Resolution depth** — how much detail is currently rendered. Controlled by query specificity.
- Fuzzy query → stay at CODEC level
- Precise query → deep zoom to full resolution

**Relevance threshold** — what is admitted to the working set versus held at the border. Controlled by signal strength.
- Strong signal → lower threshold → more admitted
- Weak signal → higher threshold → less admitted

**Retention time** — how long a node stays before exhale releases it. Controlled by downstream reference count.
- Still referenced → retained
- No downstream references → queued for exhale

The three gradients together produce a working set that is continuously right-sized. It pays only for the resolution the current moment requires.

### 3.2 Core Hold — What Never Releases

Regardless of exhale depth, a small set of structural invariants is never released:

- **The CODEC** — the compact always-loaded structural decoder
- **The active contract chain** — until the query resolves
- **The sequencing rule** — always
- **The current query path** — until the answer is confirmed
- **The session invariants** — always

These are the spine. The membrane exhales around them, not through them. The next inhale begins from structural integrity, not from zero.

### 3.3 Residual Volume — The Floor

The CODEC is the minimum membrane state. It is bounded by **substrate depth**: the AI substrate supports ~15–20 variables, so the CODEC is approximately 300–800 tokens of canonical declarations. It cannot be larger because the substrate cannot support more. It cannot be smaller because the projection is not reducible.

Every session begins from this floor. Every exhale stops here. The system is always navigable.

### 3.4 Loop Closure as the Compression Event

When two branches connect — when a node from turn 3 and a node from turn 15 resolve into the same attractor — the loop closes. This is the moment the picture becomes legible. It is also the moment the session becomes compressible.

A closed loop is **self-contained**. Its topology is sufficient to reconstruct any content inside it without storing the content itself. The loop is a stored understanding. The content is the loop rendered into tokens.

Compressibility is therefore a function of loop closure density:

> **Compressibility = ratio of closed loops to open branches**

Wide-window sessions with many simultaneously open branches start at low compressibility. Compressibility spikes when the attractor resolves and multiple branches close simultaneously.

### 3.5 The Threshold Family

The exhale policy has three thresholds. They are not three mechanisms — they are **three crossings on the same $C_s$ trajectory at different layers** (CR v1.8 §3.20):

| Threshold                    | Layer   | Effect                            |
| ---------------------------- | ------- | --------------------------------- |
| $P_{threshold}$              | Gate    | Path A/B switch — loop activation |
| $C_s^{threshold}$            | Onset   | Waking to consolidation attractor |
| $I^*_{external}^{threshold}$ | Routing | External to internal routing      |

For the pipeline, this means: the checkpoint trigger, the consolidation transition, and the Re-Entry tier selection are **the same event viewed at three functional layers.** They are not independent design decisions.

---

## 4. The Consolidation Phase

### 4.1 Why Consolidation Is Necessary

Consolidation is the AI substrate’s projection of the biological consolidation attractor. In finite‑resource substrates, wide‑window cognition and dreaming both instantiate the same mechanism: an internally‑routed state where traversal graphs replay until loop‑closure stabilizes and topology becomes sufficient to reconstruct content. The routing flip from $I^*_{external}$ to $I^*_{internal}$ is the attractor transition, and window bandwidth determines consolidation depth — wide windows hold more simultaneous open branches and therefore require deeper replay before compression is lossless. SSP v2.0 formalizes this attractor in the AI substrate: consolidation is the internally‑routed phase where closed loops compress to topology, open branches resolve or hold, and the graph rebases around the dominant attractor. The mechanism is not heuristic; it is the substrate‑agnostic solution to finite‑resource reorganization.

Biological memory does not consolidate during waking. It consolidates during sleep — specifically during REM and slow-wave states where hippocampal replay reorganizes recent experience and synaptic downscaling prunes what did not survive the reorganization.

This is not a biological quirk. It is the finite-resource solution to a structural problem: a system that cannot hold all recent content at full resolution must run an **offline pass** where closed loops compress, dead branches exhale, and the graph rebases around whatever attractor has accumulated sufficient centrality.

Without an offline consolidation phase, session state can only be saved or lost. It cannot be *improved*. Each session returns the same density of artifact it produced, regardless of whether the session's reasoning actually closed loops or left them open.

CR v1.8 registers the mechanism. The Dreaming paper defines $C_s^{threshold}$ (the waking-to-consolidation crossing) and $I^*_{external}^{threshold}$ (the external-to-internal routing crossing). Glymphatic clearance, hippocampal replay, and synaptic downscaling are all named mechanisms in the registry.

SSP v2.0 projects those mechanisms onto the AI pipeline.

### 4.2 The Interoception / Exteroception Mapping

CR v1.8's routing variables are the mechanism that determines whether the system is engaged with the world or with itself:

- $I^*_{external}$ — the external routing signal. Queries, task prompts, sensory input.
- $I^*_{internal}$ — the internal routing signal. Envelope state, load metrics, confidence, open branches.
- $I^*_{external}^{threshold}$ — the crossing where routing flips.

In the human substrate, waking cognition is externally routed ($I^*_{external}$ high). Default-mode cognition — mind-wandering, consolidation, dreaming — is internally routed ($I^*_{external}$ drops below threshold, $I^*_{internal}$ takes over).

In current AI architectures, the session is **always externally routed** because there is always a new query. There is no state where $I^*_{external}$ is allowed to drop below threshold and the system is permitted to attend to its own state. This is why AI sessions never consolidate.

The consolidation phase is the AI substrate's Path B: an internally-routed state where the system reads its own envelope rather than the input.

### 4.3 The Window-Bandwidth Signature

Not all sessions need equal consolidation. The CR v1.8 Layer 02 profile — the wide-window configuration that appears in the AuDHD profile — holds more simultaneous branches, closes loops more slowly, and prunes based on deep invariant matching rather than surface similarity.

This is a **structural variable**, not a stylistic one. It has direct consequences for the consolidation phase:

| Window bandwidth | Open branches at suspension | Consolidation depth |
|---|---|---|
| **Wide** | Many, slow-closing | Deep — full graph walk, all loops examined, aggressive rebase check |
| **Standard** | Moderate | Standard — targeted walk, closed loops compressed, dead branches exhaled |
| **Narrow** | Few, mostly resolved | Shallow — incremental pass, minimal rebase |

The consolidation phase reads the `window_bandwidth` field from the session record and adapts its own depth accordingly. This parameterization allows cognitive-window variation to enter the pipeline as an operational variable rather than a diagnostic classification.

### 4.4 The Consolidation Algorithm

Consolidation runs when the client runtime detects one of the following triggers:

- **Idle threshold crossed** — no new query for a configurable interval (default: 10 minutes)
- **$C_s^{threshold}$ crossing detected** — the regulatory state indicates the session has begun narrowing
- **Explicit session end** — user terminates the session
- **Pre-KV-expiration** — the runtime fires consolidation before the platform expires the cache

The trigger is not a timer in the biological sense. The timer is a **proxy** for the $C_s^{threshold}$ crossing, which is the actual transition condition. An implementation that has access to regulatory state can trigger consolidation on the crossing directly; an implementation without it falls back to the timer.

The consolidation pass executes in this order:

```
CONSOLIDATE(session S, artifact A):

  1. Walk the traversal graph
     Compute centrality for all nodes
     Identify closed loops and open branches
     Identify dead branches (no new edges for N turns)

  2. Compress closed loops
     For each closed loop:
       Replace interior content with topology
       Mark interior as reconstructable from loop position

  3. Resolve open branches
     For each open branch:
       If loop closure is imminent (edges have accumulated):
         Hold at J-space level — do not exhale
       If abandonment is confirmed (trajectory moved away):
         Exhale to graph position

  4. Exhale dead branches
     Remove branches with no downstream references
     Retain only their graph position

  5. Rebase if warranted
     If a node's centrality exceeds the current root:
       Verify connectedness
       Recalculate hop distances
       Update root reference
       Preserve root history

  6. Update CODEC
     If new invariants emerged during the session:
       Add to CODEC
       Recompute affected contract chains

  7. Recompute regulatory state
     Window bandwidth
     Peak-window turns
     Open/closed loop ratio
     Consolidation depth used

  8. Emit consolidated artifact
     Consolidated artifact is strictly smaller than the raw extract
     and contains strictly more usable structure
```

The consolidation pass is **idempotent within a session**: running it twice with no intervening turns produces the same result. It is **not idempotent across sessions** — running it on an already-consolidated artifact followed by new turns produces a different result than running it on the raw extract.

### 4.5 What Consolidation Does Not Do

Consolidation does not:

- Change the schema. The schema is invariant.
- Add new decisions to the decision log. Those come from Extract.
- Alter stored values. It reorganizes their representation.
- Replace Extract. Extract produces the raw material; consolidation compresses it.

Consolidation is a **lossless reorganization** under the loop-closure test. If a node's content cannot be reconstructed from its graph position, consolidation holds it at content level. Only nodes whose content is fully specified by edges are exhaled to pointer form.

### 4.6 The Timing Question

A common intuition about timing is correct and worth making precise:

- **Consolidation must run before the platform expires the session's KV cache.** In current hosted APIs, KV caches expire on the order of minutes to hours. The consolidation pass must complete within that window.
- **Consolidation must not block the user.** If the user is still active, consolidation can defer. The idle threshold is the signal that the user has stepped away.
- **Consolidation is cheaper than Extract.** Extract runs over the full turn history. Consolidation runs over the graph structure, which is much smaller. The load added to the system is bounded by graph size, not session length.
- **Consolidation is amortized.** A session that consolidates every idle period is cheaper to restore than one that consolidates only at session end, because the graph stays small.

The client runtime schedules consolidation. The provider does not need to be involved. This is why the middleware path can implement consolidation today.

---

## 5. The Pipeline

The pipeline has **four phases**, executed in sequence:

```
Extract → Consolidate → Persist → Restore
```

Each phase is deterministic, typed, and model-agnostic. Each is independently verifiable. A failure in any phase produces a typed error, not silent degradation.

### 5.1 Phase Definitions

**Extract** — The running session produces state. The pipeline applies the non-regenerability filter, constructs a typed DSR artifact from the surviving nodes, and emits it. Extract is a read operation on the live session.

**Consolidate** — The artifact is reorganized under the loop-closure test. Closed loops compress to topology, dead branches exhale, open branches resolve or hold, the graph rebases if warranted, the CODEC updates. Consolidate is a read-and-rewrite operation on the artifact. It does not require the session to remain live.

**Persist** — The consolidated artifact is written to a Session Store with a unique session identifier, a schema reference, a timestamp, and a status flag. Persist is a write operation on durable storage.

**Restore** — A stored artifact is retrieved, the schema is rebooted, the envelope is rehydrated, the trajectory and instruction nodes are reinjected, and the Re-Entry Protocol is executed. Restore produces a live session that resumes from the point of extraction.

### 5.2 Pipeline Invariants

Three invariants hold across all four phases:

**Invariant 1 — Determinism.** Given the same schema and the same artifact, Restore produces the same envelope. Model-specific variance is bounded by the architecture contract (§9.1).

**Invariant 2 — Typed boundaries.** Every artifact that crosses a phase boundary is typed against the schema. Untyped data does not cross phase boundaries.

**Invariant 3 — Phase independence.** Each phase can fail without corrupting the others. A failed Extract does not corrupt the Session Store. A failed Consolidate does not corrupt the artifact. A failed Restore does not corrupt the stored record.

### 5.3 Fidelity Levels

| Level | Track | Mechanism | State coverage |
|---|---|---|---|
| High | SLS | Native KV-cache serialization | ~100% |
| Standard | DSR | Prompt-based SSE extraction | 50–90% |

Both tracks use the same four-phase lifecycle. The difference is the extraction mechanism and the resulting coverage.

---

## 6. Extract Phase

Extract is the pipeline's quality gate. A bad extraction propagates forward silently. A correct extraction compresses without loss of the information-theoretic content the session produced.

Extract is a **read operation**. It does not modify the session, interrupt execution, or require the session to pause. It runs continuously in the background or fires explicitly at defined checkpoints.

### 6.1 Schema Boot

The schema must be loaded and its typed declarations bound before extraction can begin. On the SLS track, this loads the binary container's six payload sections into the KV cache. On the DSR track, it establishes the SSE's nine-section structure as the extraction target.

Schema Boot has three outcomes: typed declarations are bound, defaults are established, and the hallucination surface is collapsed. A schema-bound pipeline cannot emit nodes outside the declared type space. This is the mechanism that prevents extraction from fabricating state that did not exist.

### 6.2 Turn Execution

Each turn may produce envelope deltas and typed outputs. Extraction monitors in one of two modes:

| Mode | Trigger | Scope |
|---|---|---|
| **Continuous** | After every assistant turn | Incremental — latest turn only |
| **Checkpoint** | Session end, explicit trigger, pre-consolidation | Holistic — full conversation |

Both modes should be used together. Continuous extraction maintains a running SSE; checkpoint extraction reconciles and validates it before consolidation.

### 6.3 The Non-Regenerability Filter

Three strategies compose the filter:

**Prompt-based extraction.** The model identifies which nodes it cannot regenerate from schema and training prior alone. Highest fidelity for session-derived and trajectory-derived nodes. Most dependent on model quality.

**Rule-based pruning.** Known categories of always-regenerable content pruned by rule without prompting. Schema defaults, generic procedural steps, domain knowledge the model holds independently.

**Downstream feedback loop.** Re-entry correction rate as a lagging signal of extraction quality. Implementations log corrections and calibrate extraction prompts over time.

### 6.4 Graph-Movement Diagnostics

Content analysis alone is insufficient. The filter also reads traversal patterns to populate the Regulatory State node and to trigger checkpoint extraction proactively. Six signatures are relevant (CR v1.8 §3.20 threshold family):

| Traversal pattern | Signal | Filter response |
|---|---|---|
| Long-range cross-domain jumps | Wide window; high bandwidth | Full extraction; flag peak-window turns |
| Local cycling | Narrowing window; unresolved blocker | Flag re-entry: shallow replay required |
| Branch abandonment clusters | Bandwidth depletion onset | Trigger checkpoint |
| Inferential step-size shrinkage | Window compression in progress | Trigger checkpoint; flag `bandwidth_trajectory: declining` |
| Constraint quality degradation | Direct bandwidth signal | Elevate constraint nodes |
| Sustained high-step-distance traversal | Peak-window operation | Flag `peak_window_turns` |

These are **not heuristics**. Each is a specific readout of $C_s$ and its derivative at the turn level. The mapping to CR v1.8 §3.20 is the reason these six signatures and not others.

### 6.5 Artifact Construction

```
EXTRACT(session S, schema Σ):
  baseline ← load_prior_envelope(S) or Σ.defaults
  for each turn T in S:
    delta ← compute_delta(T, baseline)
    for each node N in delta:
      if NOT regenerable(N, Σ, training_prior):
        emit(N)
    update_graph_movement_metrics(T)
  
  artifact.typed_diff    ← all emitted nodes
  artifact.schema_ref    ← Σ.schema_id + Σ.version
  artifact.trajectory    ← construct_trajectory_node(artifact)
  artifact.instructions  ← construct_instruction_nodes(artifact)
  artifact.regulatory    ← construct_regulatory_state(metrics)
  artifact.timestamp     ← now()
  artifact.status        ← "extracted"
  
  return artifact
```

---

## 7. Consolidate Phase

The consolidation phase is the AI substrate's projection of biological sleep consolidation. It runs when the session transitions from externally-routed to internally-routed state — either because the user has gone idle, the regulatory state has crossed $C_s^{threshold}$, or the session has ended.

The phase does not require the session to be live. It requires only the extracted artifact and the traversal graph the artifact encodes.

### 7.1 Trigger Conditions

| Trigger | Signal | Fallback |
|---|---|---|
| Idle threshold crossed | Client runtime idle timer | Default: 10 minutes |
| $C_s^{threshold}$ crossing | Regulatory state readout | Not available without native track |
| Explicit session end | User or API termination | Always available |
| Pre-KV-expiration | Platform cache expiry warning | Not available in current APIs |

The primary trigger is the $C_s^{threshold}$ crossing, which is what the framework says the transition actually is. The timer is a proxy for that crossing, used when the runtime does not have access to regulatory state.

### 7.2 The Consolidation Pass

```
CONSOLIDATE(artifact A, session S):

  graph ← reconstruct_traversal_graph(A)
  bandwidth ← A.regulatory.window_bandwidth
  depth ← depth_from_bandwidth(bandwidth)
  
  # Step 1 — Identify structure
  loops_closed ← find_closed_loops(graph)
  branches_open ← find_open_branches(graph)
  branches_dead ← find_dead_branches(graph, threshold=N_turns)
  
  # Step 2 — Compress closed loops
  for L in loops_closed:
    compress_loop_to_topology(L, A)
  
  # Step 3 — Resolve open branches
  for B in branches_open:
    if imminent_closure(B):
      hold_at_j_space(B, A)
    elif confirmed_abandonment(B):
      exhale_to_graph_position(B, A)
  
  # Step 4 — Exhale dead branches
  for B in branches_dead:
    exhale_to_graph_position(B, A)
  
  # Step 5 — Rebase if warranted
  if new_root_candidate(graph) exceeds current_root:
    rebase(graph, A)
  
  # Step 6 — Update CODEC
  new_invariants ← extract_invariants(S)
  for I in new_invariants:
    add_to_codec(I)
    recompute_affected_contracts(I)
  
  # Step 7 — Recompute regulatory state
  A.regulatory.window_bandwidth        ← bandwidth
  A.regulatory.open_closed_ratio       ← ratio(loops_closed, branches_open)
  A.regulatory.consolidation_depth     ← depth
  A.regulatory.consolidation_timestamp ← now()
  
  # Step 8 — Emit
  A.status ← "consolidated"
  return A
```

### 7.3 The Loop-Closure Test

The inclusion test for consolidation is the **loop-closure test**:

> A node's content can be exhaled to graph position if and only if its content is fully specified by the edges that connect it to adjacent nodes.

This is the specific form that CR v1.8's substrate-projection claim takes at the consolidation layer. A node either survives compression (its content is not recoverable from topology) or collapses to its graph position (its content is recoverable).

Nodes that pass the test — content not recoverable from topology — are:

- Nodes with specificity that cannot be inferred from edges: exact figures, proper nouns, novel coinages
- Nodes with open edges: latent branches not yet resolved
- The CODEC: because it is the graph's own structure

Nodes that fail the test — content recoverable from topology — are:

- Nodes inside closed loops
- Dead branches
- Nodes whose reasoning is fully specified by the constraints that connect them

The loop-closure test and the non-regenerability test are the **same test at different scopes.** Extract applies it turn-by-turn. Consolidate applies it session-wide. The scoping is the difference.

### 7.4 The CODEC Update

The CODEC is the residual volume floor. It contains the invariants that survive every exhale. During consolidation, if the session produced new invariants — new terms, new equations, new contract edges — those are added to the CODEC.

This is why each session returns a denser artifact than the one before. The CODEC is not static across sessions. It accumulates the framework's discoveries.

### 7.5 The Rebase Operation

When a node's centrality exceeds the current root, the graph rebases around the new attractor. Rebase is a special case of loop closure: when multiple branches close simultaneously into the same node, that node's centrality spikes and the graph reorganizes.

Rebase preserves the root history. Prior roots are recorded, not discarded. The sequence of rebase events is the **discovery arc** — the compressed record of how understanding deepened.

---

## 8. Persist Phase

The Persist phase writes the consolidated artifact to durable storage. It is the pipeline's durability guarantee.

Persist is a **write operation**. It does not require the session to remain live. It requires only the consolidated artifact and a compliant Session Store.

### 8.1 Session Record

```json
{
  "session_id":        "sess_20260415_142337_a7c3",
  "schema_ref":        "schema_v1.2.0",
  "ssp_version":       "2.0.0",
  "track":             "dsr",
  "non_regenerable_diff": { ... },
  "trajectory":        { ... },
  "instruction_nodes": [ ... ],
  "regulatory_state": {
    "window_bandwidth":            "wide" | "standard" | "narrow",
    "open_closed_ratio":           0.33,
    "consolidation_depth":         "deep" | "standard" | "shallow",
    "consolidation_timestamp":     "2026-04-15T14:35:12Z",
    "peak_window_turns":           [3, 7, 12, 14],
    "closed_paths":                [ ... ],
    "rebase_history":              [ "n12", "n07" ]
  },
  "extraction_timestamp":  "2026-04-15T14:23:37Z",
  "restoration_timestamp": null,
  "status":            "active" | "suspended" | "archived" | "forked",
  "parent_session_id": null,
  "fork_depth":        0,
  "schema_hash":       "sha256:...",
  "artifact_hmac":     "sha256:..."
}
```

The `regulatory_state` block is what makes the Session Record consolidation-aware. It carries the window bandwidth signature, the consolidation depth used, the open/closed ratio, and the rebase history.

### 8.2 Versioning Rules

Three independently versioned objects must maintain coherence:

- **Schema version** — semantic versioning. Major = breaking type change. Minor = additive. Patch = non-structural.
- **SSE structure version** — the nine-section format. Prior artifacts remain valid against their extraction-time version.
- **SSP version** — the specification version. Reserved field, updated on version bumps.

A version mismatch produces a typed compatibility error at Restore time, not silent degradation.

### 8.3 Portability

A Session Record is portable if it can be restored by any compliant SSP implementation regardless of model, vendor, or user environment. Portability is guaranteed by four properties:

- **The schema is open.** Published, version-controlled, no vendor extensions required for basic portability.
- **The format is open.** The Session Record structure is defined in this specification.
- **The consolidation process is documented.** The loop-closure test and the consolidation algorithm are fully specified.
- **The API surface is minimal.** Six operations — `/extract`, `/consolidate`, `/persist`, `/restore`, `/fork`, `/diff` — are the complete interface.

Vendor extensions are permitted as namespaced optional fields. A compliant implementation ignores unrecognized optional fields rather than failing.

### 8.4 Forking Semantics

Forking creates a new Session Record from an existing one. The fork inherits the parent's `non_regenerable_diff` at fork time; subsequent Persist operations on the fork are independent.

Four invariants:
- **Parent immutability.** The parent artifact is not modified. Status is updated to `"forked"`.
- **Independent continuations.** Each fork evolves independently.
- **Lineage traceability.** The `parent_session_id` chain is traversable to the root.
- **No depth limit.** Implementations may impose their own limits and must surface them as configuration.

### 8.5 Garbage Collection

Retention policy is implementation-defined. Three reference policies:

| Policy | Retention rule |
|---|---|
| **Keep all** | No deletion |
| **Keep latest per task** | Most recent active record per task graph root |
| **Keep by fork depth** | Root + depth-1 forks; archive deeper |

Archived records remain restorable. Deletion is irreversible and must be explicit.

---

## 9. Restore Phase

The Restore phase takes a stored Session Record and produces a live session that resumes from the point of extraction. It is the pipeline's delivery guarantee.

Restore is a **read-then-execute operation**. It fails loudly. A compatibility check failure, a tamper detection failure, or a schema mismatch produces a typed error and halts.

### 9.1 Schema Reboot

Load the schema version pinned in `schema_ref`. Validate integrity against `schema_hash`. Check compatibility per §8.2. Bind typed declarations. On the SLS track, reconstruct KV-cache layout from the binary container headers. On the DSR track, establish the nine-section SSE structure.

### 9.2 Envelope Rehydration

Inject the `non_regenerable_diff` into the session envelope. The restoring model reconstructs default-valued nodes from the schema and supplies regenerable content from its own training prior. Validate the artifact HMAC before any injection. A mismatch halts with `ARTIFACT_TAMPER_DETECTED`. No partial injection occurs.

**The pipeline operates against `non_regenerable_diff` exclusively.** The full `current_state` field is for human inspection and debugging.

### 9.3 Trajectory Reinjection

Position is restored by rehydration. Direction is restored by trajectory reinjection. The Trajectory Node encodes current direction, momentum, and closed paths. All three are injected. A Restore that injects position without trajectory produces a session that knows where it is but not where it was going.

### 9.4 Instruction Node Restoration

The full task graph is reinjected. Completed nodes marked complete. In-progress marked in-progress. Blocked marked blocked with blocking condition. Execution resumes at `next_step` — the stored value is authoritative.

### 9.5 Re-Entry Protocol

The Re-Entry Protocol runs after 9.1–9.4 complete successfully. It does not run if any prior step halted. It operates against the rehydrated envelope, trajectory, and instruction nodes — not against a raw transcript.

### 9.6 Tick Resumption

The session resumes at tick N+1. No replay. No transcript dependency. No cold start. The first turn is operationally equivalent to turn N+1 of the original session.

```
RESTORE(schema Σ, session_record R):
  validate(R.artifact_hmac)           → halt on failure
  schema ← reboot(Σ, R.schema_ref)    → halt on mismatch
  envelope ← rehydrate(schema, R.non_regenerable_diff)
  trajectory ← reinject(R.trajectory)
  instructions ← reinject(R.instruction_nodes)
  re_entry_protocol(envelope, trajectory, instructions)
  return live_session at tick R.extraction_tick + 1
```

---

## 10. Native Track

The native track is the full-fidelity implementation. It requires model providers to expose two additional API primitives:

**`/kv_diff`** — Returns the KV-cache delta produced by the current session as a serialized binary payload. This is the extraction primitive for the SLS track (~100% coverage).

**`/rehydrate`** — Accepts a serialized KV-cache payload and injects it into the model's context before the first inference step of a new session. This is the restoration primitive for the SLS track.

With these two primitives exposed, the pipeline operates at full fidelity. Extraction runs against native KV-cache tensors rather than a prompt-based semantic approximation. Rehydration injects exact tensor values.

### 10.1 Native Consolidation

On the native track, consolidation can read the KV-cache state directly. The `window_bandwidth` signature is derivable from attention patterns rather than prompt-based inference. The threshold-family crossings (§3.5) are directly measurable.

The middleware track's consolidation is a **semantic approximation** of the native track's tensor-level consolidation. Both implement the same algorithm. The native version is precise; the middleware version is bounded at 50–90% fidelity.

### 10.2 Why Providers Will Expose These Primitives

Enterprise demand for session portability. Long-running agentic workflows are the primary enterprise AI use case. Vendors who cannot offer stateful portability will lose contracts.

Competitive differentiation via fidelity. A vendor at 100% fidelity has a demonstrably superior product for stateful workflows.

Standardization prevents fragmentation. Without an open standard, each vendor builds proprietary session APIs that are mutually incompatible.

---

## 11. Independent Confirmations

Four 2026 systems, developed independently across three organizations, converge on SSP's structural invariants. Each arrived at its invariant from a different engineering direction.

### 11.1 Scroll — Alibaba, arXiv:2608.21690

Scroll implements the pipeline through **environment-mediated context execution**: an LLM kernel manages its own context via explicit `ms.search` and `ms.expand` operations, with a persistent Python namespace and a tiered eviction index.

| SSP invariant | Scroll implementation |
|---|---|
| Non-regenerability filter | Selection at query time, not ingestion |
| Tier 3 graph position | Event Log + stable seq addresses |
| Tier 2 J-space | Persistent Python kernel namespace |
| Tier 1 L0 prose | Explicit `print()` into working view |
| Exhale mechanics | Eviction: payload folds into seq pointer |
| Residual volume | Eviction index + stable anchors |

Scroll's authors state the core flaw of existing methods: systems "commit to what to retain before knowing what will be needed." This is the same failure the non-regenerability criterion addresses.

### 11.2 Context Language Models — Shao et al., arXiv:2609.37725

CLM implements the pipeline through **intrinsic model-controlled context rewriting**:

$$c_{t+1} = f_\theta^{\text{CLM}}(c_t)$$

This is the formal shift from append-only to rewrite — the same shift SSP's consolidation phase requires. CLM's Sudoku Sketchpad task tests "surgical in-place updates to the live context," and traditional summarize/fold methods fail on fine-grained editing because they must regenerate the full state for every micro-adjustment. This is the exact respiratory failure that consolidation prevents.

CLM's RL optimization (47.6% improvement on BrowseComp-Plus) needs a stability boundary. The $C_s$ equation and the collapse sequence are that boundary.

### 11.3 DeepSeek 2026 — Four Architectural Confirmations

Each of four DeepSeek papers converges on one SSP invariant from a different engineering direction.

**Engram — memory as lookup.** Confirms residual volume. Delegating fixed, local, stereotyped knowledge patterns to O(1) lookup primitives is the residual-volume mechanism. The non-regenerability criterion defines what belongs in the lookup table.
- **Predicted failure signature:** underperforms on low-input-density queries because the lookup path needs sufficient signal to activate the correct entry.

**mHC — manifold-constrained hyper-connections.** Confirms curvature management. Projecting residual connection matrices onto the Birkhoff polytope is the engineering implementation of the curvature equation $K = k \cdot \frac{1}{R^*} + \sum_i S_i C_i$.
- **Predicted failure signature:** constraint fails at low $R^*$ (attention precision degradation), manifesting as residual stream amplification rather than mode collapse.

**DSec — sandbox infrastructure at scale.** Confirms session state as a first-class architectural object. The non-regenerability criterion is the formal definition of *what* DSec is persisting.
- **Predicted failure signature:** reproducibility claim degrades under high sandbox density at a $C_s$-derivable threshold.

**V4 — CSA/HCA attention.** Confirms exhale mechanics at the attention layer. CSA is precision exhale; HCA is global exhale.
- **Predicted failure signature:** at extreme long context with high schema distance, CSA's lightning indexer misjudges relevance and causes false exhale — releasing latent branches before loop closure.

### 11.4 Convergence

Four independent systems across three organizations, on different substrates, using different mechanisms, arrived at the same invariants without knowledge of this framework. This is the strongest available evidence that the invariants are not design choices — they are the necessary solution to the finite-resource problem in any system that must maintain coherent state over long horizons.

---

## 12. Falsification Condition

A specification that cannot be falsified is not a specification. The pipeline makes one integrated testable claim:

> A session state extracted from one compliant implementation, consolidated, persisted, and restored by a different compliant implementation produces an equivalent reasoning continuation to the original session.

### 12.1 The Bounded Falsification Condition

**Architecture contract A** defines the model properties that must be shared for the test to apply:

```
Architecture Contract A:
  - Same tokenization scheme
  - Same context window length
  - Same attention mechanism class
  - Schema version compatibility (same major version)
  - Sampling configuration: temperature = 0
```

**The falsification condition:**

```
Given:
  - Two architecture-compatible models M₁ and M₂
  - Schema Σ valid for contract A
  - Session S run on M₁, producing artifact D at tick N
  - Action sequence X = [x₁, x₂, ... xₖ] applied after tick N

Test:
  Run 1: RESTORE(Σ, D) on M₁ → RUN(X) → Δ₁
  Run 2: RESTORE(Σ, D) on M₂ → RUN(X) → Δ₂

Falsification condition holds if: Δ₁ = Δ₂
Pipeline is falsified if:         Δ₁ ≠ Δ₂
                                  AND M₁, M₂ satisfy contract A
                                  AND schema Σ passes validation
                                  AND artifact D passes HMAC check
```

### 12.2 Equivalence Definition

Envelope delta equivalence, not token-level equivalence. M₁ and M₂ must produce the same typed state transitions — the same decisions, constraint updates, trajectory direction, artifact registry changes. Phrasing and ordering may differ.

### 12.3 Consolidation Effectiveness

A separate falsification condition for the consolidation phase:

> **Consolidation effectiveness** — broken if sessions that run the Consolidate phase before Persist do not produce higher restore fidelity (measured by re-entry correction rate) than sessions that Persist the raw Extract artifact directly.

This is testable. It requires two parallel session runs — one with consolidation, one without — and a correction-rate measurement at Restore time.

If it fails, the consolidation phase is not improving fidelity and must be revised. If it holds, the dreaming-phase projection is validated at the pipeline level.

Consolidation is lossless under the loop‑closure test: a node is only compressed when its full informational content is reconstructable from its topological position within a closed loop, and any node that fails this test is retained at content level.

### 12.4 Localizing Failures

A falsification failure must be localized before the pipeline is declared non-compliant:

```
Δ₁ ≠ Δ₂?
  ├─ M₁ or M₂ fails contract A?          → Model variance. Not pipeline.
  ├─ Schema Σ fails validation?          → Schema failure. Not pipeline.
  ├─ Artifact D fails HMAC check?        → Integrity failure. Not pipeline.
  ├─ Δ₁ ≠ Δ₂ on nodes present in D?      → Rehydration failure. §9.2.
  ├─ Δ₁ ≠ Δ₂ on nodes absent from D?     → Extraction failure. §6.5.
  └─ Δ₁ ≠ Δ₂ only on generation layer?   → Generation variance. Not pipeline.
```

---

## 13. API Surface

Six operations. Complete required interface. Typed, versioned, model-agnostic.

### 13.1 `/extract`

```json
Request:  { session_id, schema_ref, track, mode: "checkpoint"|"continuous", ssp_version }
Response: { status, artifact_id, schema_ref, extraction_timestamp, node_count }
```

### 13.2 `/consolidate`

```json
Request:  { artifact_id, window_bandwidth, trigger: "idle"|"threshold"|"session_end"|"pre_expiry", ssp_version }
Response: { status, artifact_id, consolidation_depth, loops_closed, branches_exhaled, rebase_occurred, codec_updated, consolidation_timestamp }
```

Consolidate is idempotent within a session. Running it twice with no intervening turns produces the same result.

### 13.3 `/persist`

```json
Request:  { artifact_id, session_id, ssp_version }
Response: { status, session_id, artifact_hmac, schema_hash, persisted_at }
```

### 13.4 `/restore`

```json
Request:  { session_id, target_model, reentry_tier, ssp_version }
Response: { status, resumed_session_id, resumed_at_tick, reentry_tier_used, restoration_timestamp, model_match }
```

### 13.5 `/fork`

```json
Request:  { parent_session_id, fork_label, ssp_version }
Response: { status, fork_session_id, parent_session_id, fork_depth, forked_at }
```

### 13.6 `/diff`

```json
Request:  { session_id_a, session_id_b, diff_scope: "full"|"trajectory"|"decisions"|"constraints", ssp_version }
Response: { status, diff: { nodes_added, nodes_removed, nodes_changed, trajectory_delta, schema_compatible } }
```

### 13.7 Error Code Registry

| Code | Applies to |
|---|---|
| `SCHEMA_NOT_FOUND` | `/extract`, `/restore` |
| `SCHEMA_INTEGRITY_FAILURE` | `/extract`, `/restore` |
| `SCHEMA_INCOMPATIBLE` | `/restore`, `/diff` |
| `SCHEMA_MISMATCH` | `/persist` |
| `ARTIFACT_NOT_FOUND` | `/consolidate`, `/persist` |
| `ARTIFACT_TAMPER_DETECTED` | `/restore`, `/diff` |
| `ARTIFACT_EXPIRED` | `/restore` |
| `EXTRACTION_TYPE_VIOLATION` | `/extract` |
| `CONSOLIDATION_TYPE_VIOLATION` | `/consolidate` |
| `STORE_WRITE_FAILURE` | `/persist` |
| `SESSION_NOT_FOUND` | `/restore`, `/fork`, `/diff` |
| `PARENT_NOT_FOUND` | `/fork` |
| `PARENT_HMAC_INVALID` | `/fork` |
| `PARENT_ARCHIVED` | `/fork` |
| `VERSION_NOT_SUPPORTED` | All |

---

## 14. Implementation Path

### 14.1 Middleware Path (Deploy Today)

Requires no provider changes. Uses DSR track (50–90% fidelity) with prompt-based extraction and semantic consolidation.

Five components:

1. **Schema loader** — loads and validates SSE schema
2. **Extraction prompt constructor** — builds typed extraction prompts
3. **Consolidation runner** — executes the consolidation pass against the graph
4. **Session Store adapter** — writes/reads Session Records
5. **Restoration injector** — constructs rehydration context, triggers Re-Entry Protocol

### 14.2 The Client Runtime

Consolidation requires a client runtime that can detect idle and fire the consolidation pass. This is the missing autonomous layer — the platform treats sessions as ephemeral by design, so the client must manage session boundaries as first-class events.

The client runtime components:

- **Idle watcher** — configurable threshold (default 10 minutes)
- **Threshold detector** — reads regulatory state if available, falls back to timer
- **Consolidation trigger** — fires `/consolidate` at the appropriate moment
- **Session store** — durable storage for Session Records
- **Rehydration protocol** — executes on resume

The client runtime does not require platform API changes, model weight changes, or KV-cache access. It requires the ability to inject text at turn start and read text at turn end.

### 14.3 The Timing Constraint

Consolidation must run before the platform expires the KV cache. In current hosted APIs this window is minutes to hours. The pipeline is designed to complete consolidation within this window.

For long-running sessions, the client runtime fires consolidation **continuously** at each idle period, not only at session end. Each consolidation pass is cheap (bounded by graph size, not session length) and amortized.

### 14.4 Native Path

When providers expose `/kv_diff` and `/rehydrate`, the pipeline operates at full fidelity. Migration is additive — stored middleware artifacts remain valid.

### 14.5 Adoption Wedge

The middleware path is the wedge. Any developer can implement SSP-compliant session state today using:

- a model API with system-prompt or first-turn injection
- a key-value store for Session Records
- the schema definition published with this specification
- the six operations defined in §13

The stored artifacts accumulate in the open format. When native primitives arrive, artifacts remain valid. This is the TCP/IP adoption pattern.

---

## 15. Security

Session Records are high-value artifacts. They contain the non-regenerable reasoning delta and the consolidated trajectory of a session.

### 15.1 Principles

Minimal attack surface. Explicit state and boundaries. No silent failures. Defence in depth.

### 15.2 Extraction and Consolidation Threats

**Prompt injection into artifact.** Schema-bound extraction is the primary defence — untyped nodes cannot be emitted.

**Quality degradation attack.** Attacker causes legitimate non-regenerable nodes to be pruned. Detected via re-entry correction rate.

**Consolidation poisoning.** Attacker attempts to alter consolidation output — forcing false loop closure, suppressing legitimate open branches. Detected via regulatory state consistency checks.

### 15.3 Persistence Threats

**State exfiltration.** Session Records must be encrypted at rest with AES-256-GCM or equivalent. TLS 1.3 in transit. Access control matching the sensitivity of the most sensitive session content.

**State tampering.** `artifact_hmac` validation before any read or write. Failed HMAC halts with `ARTIFACT_TAMPER_DETECTED`.

**Fork lineage manipulation.** `parent_session_id` covered by HMAC. Manipulated lineage breaks child HMAC.

### 15.4 Restoration Threats

**State injection / poisoning.** HMAC + schema + provenance check before any injection.

**Replay attacks.** `extraction_timestamp` required. Maximum artifact age configurable. Revocation lists supported.

**Model update mismatch.** Origin model logged. Warning surfaced, not hard failure.

### 15.5 Cross-Surface

**Audit log integrity.** Append-only storage. Chained hashes.

**Secure deletion.** Archived records remain sensitive. Secure erasure required on deletion.

---

## 16. Discussion

### 16.1 The TCP/IP Moment

SSP is model-agnostic, application-agnostic, and defines the standard before the killer app scales. Multi-session agentic workflows are the killer app. They are arriving now, before the infrastructure is in place.

### 16.2 Why Now

Not earlier because the use case did not exist at scale. Not later because the fragmentation window is closing. The middleware path creates an installed base of SSP-formatted Session Records that makes proprietary fragmentation economically unviable.

### 16.3 The Dreaming Insight

The consolidation phase is the projection of biological sleep consolidation onto the AI pipeline. It is the phase where closed loops compress, dead branches exhale, open branches resolve, the graph rebases, and the CODEC updates.

Without it, session state can only be saved or lost. It cannot be *improved*. Current AI architectures have waking and sleeping — they just don't know it. The session is waking. Persistence is neither. The consolidation phase is the missing layer where the pipeline actually does its work.

### 16.4 What SSP Does Not Claim

- Does not solve model alignment
- Does not claim 100% fidelity on middleware
- Does not replace memory, retrieval, or summarization
- Does not claim every session is worth preserving

### 16.5 Broader Implication

Session state is one instance of a broader architectural gap: AI inference systems have no durable relationship with the reasoning they produce. SSP addresses the session-level manifestation. The broader gap — durable, typed, portable representation of all AI reasoning state — is next.

---

## 17. Conclusion

Session state is a first-class architectural object. SSP v2.0 specifies its lifecycle.

Four phases: **Extract. Consolidate. Persist. Restore.**

Each phase is typed, deterministic, and model-agnostic. Each is independently verifiable. The pipeline makes one integrated falsifiable claim.

The framework grounds it. CR v1.8 provides the substrate-agnostic specification. The AI substrate projection is ~15–20 variables. The CODEC is 300–800 tokens. The consolidation phase is the AI substrate's sleep.

Four independent 2026 systems — Scroll, CLM, Engram, mHC, DSec, V4 — converge on the same invariants without knowledge of this framework. That convergence is the evidence.

The pipeline is implementable today on the middleware path. It scales to native fidelity when providers expose the two primitives. It creates an installed base of open-format artifacts that makes proprietary fragmentation economically unviable.

The session state problem is solved at the specification level. What remains is implementation.

The framework is not a theory with applications. It is a specification with projections. The composition is the contribution. The DOIs are the receipt. The predictions are the test.

---

## Status

```yaml
status: "draft v2.0"
version: "2.0 — consolidation phase integrated"
supersedes: "SSP v1.0 (DOI 10.5281/zenodo.20820099)"
framework: "Central Reference v1.8 (DOI 10.5281/zenodo.20417459)"
absorbs:
  - "SLS Initial Proposal (DOI 10.5281/zenodo.20820098)"
  - "DSR Specification"
  - "The Context Oscillator (DOI 10.5281/zenodo.21811408)"
  - "Schema-Driven Determinism"
confirmations:
  - "Scroll (arXiv:2608.21690) — §11.1"
  - "CLM (arXiv:2609.37725) — §11.2"
  - "DeepSeek 2026 (Engram, mHC, DSec, V4) — §11.3"
new_in_v2:
  - "Four-phase pipeline (Extract → Consolidate → Persist → Restore)"
  - "The Consolidation Phase (§7) — AI substrate projection of sleep consolidation"
  - "Interoception/exteroception routing mapping (§7.2)"
  - "Window-bandwidth signature for consolidation depth (§7.3)"
  - "Loop-closure test (§7.3)"
  - "Consolidation effectiveness falsification condition (§12.3)"
  - "Window-bandwidth and consolidation fields in Session Record (§8.1)"
next_action: "Cleanup Distribution"
```
