# Serializable Latent State
*Initial Proposal for LLM Save Files*

| Field | Value |
| --- | --- |
| **Version** | v0.1-draft |
| **Date** | April 2026 |
| **Author** | Working Group — Initial Draft |
| **Status** | DRAFT — FOR DISCUSSION |
| **Audience** | Systems Architects, AI Infrastructure Engineers, Technical Leadership |

---

## 1. Abstract

This proposal introduces **Serializable Latent State (SLS)** — a new systems-level primitive that enables the complete computational context of a large language model session to be captured, persisted, transmitted, and restored as a portable binary artifact. LLM sessions today are fundamentally ephemeral: when a session ends, the rich internal representation the model has built — its KV-cache, sampling state, and accumulated attention patterns — is destroyed irreversibly, and reconstructing it from a transcript is both lossy and computationally expensive. SLS solves this by defining a standardized, versioned, opaque-but-structured snapshot format (informally, an "LLM save file") that freezes the model's latent state at a point in time, enabling instant resume, branching, migration, and forensic audit. The primitive is infrastructure-grade: it targets the same layer of the stack as container images, memory snapshots, and process checkpoints — not application-level features like chat history export.

---

## 2. Concept Definition

A **Serializable Latent State (SLS)** is a portable, versioned, opaque-but-structured snapshot of the complete computational context required to deterministically resume an LLM session at a prior point. It is not a log of what was said. It is a capture of what the model *currently understands* — the internal numerical state from which the next token would be generated.

An SLS artifact encapsulates the following components:

- **KV-Cache** — The attention key and value tensors for all transformer layers. This is the dominant component by size and the most critical by function. It encodes the model's accumulated "understanding" of the entire token sequence processed so far, including cross-token attention relationships that cannot be inferred from the token sequence alone.
- **Token History / Prompt Buffer** — The complete sequence of token IDs currently within the model's context window. This serves as a verification reference and enables prompt reconstruction for display purposes, but it is *not* the primary state — the KV-cache is.
- **Sampling State** — The random number generator seed and current RNG position, temperature, top-p, top-k, repetition penalty, and any other sampling hyperparameters active at the moment of capture. Restoring these ensures that the next token sampled from a deserialized state is identical to what would have been sampled had the session continued.
- **System Prompt and Instruction Bindings** — A cryptographic hash of the system prompt and any instruction-layer configuration (e.g., tool definitions, function schemas, behavioral constraints) active at capture time. This enables verification that the deserialized state is being resumed under compatible instructions.
- **Adapter / LoRA Weights** — If any parameter-efficient fine-tuning adapters (LoRA, QLoRA, or equivalent) were active at capture time, their delta weights are included. The SLS must reproduce the exact model behavior, including adapter influence on attention and feedforward computations.
- **Metadata Envelope** — A structured, human-readable block containing: model identity (name, family, version), architecture hash (a deterministic fingerprint of the model's structural parameters), weights hash, timestamp, session ID, provenance chain (parent SLS hash, if any), creator identity, and free-form tags or descriptions.

### 2.1 What SLS Is Not

Clarity requires distinguishing SLS from adjacent concepts that superficially resemble it:

| Concept | What It Captures | Key Difference from SLS |
| --- | --- | --- |
| **Conversation Log** | The text of user and assistant messages | Captures *inputs and outputs*, not internal state. Replaying a log does not reproduce the same KV-cache due to floating-point non-determinism, batching differences, and quantization artifacts. |
| **RAG Retrieval Snapshot** | The documents retrieved and injected into context | Captures *external knowledge at retrieval time*, not the model's processed representation of it. Two identical retrievals processed in different batch configurations yield different latent states. |
| **Model Checkpoint** | The model's trained weights at a point in the training process | Captures the *model itself*, not a session. A checkpoint is the instrument; an SLS is a recording made with that instrument at a specific moment. |
| **Context Summary / Compression** | A condensed natural-language summary of prior conversation | Captures a *lossy approximation* of context. Summaries discard attention patterns, token-level relationships, and latent representations that the model has built. |

### 2.2 The Save File Analogy

The informal term "LLM save file" is deliberate. A video game save file does not store every button press the player ever made — it stores the *world state* at a moment in time: character position, inventory contents, quest flags, NPC states, environmental variables. Loading a save file does not replay the game from the beginning; it reconstructs the world exactly as it was and lets the player continue immediately. SLS applies this same principle to LLM inference. The KV-cache *is* the world state. The token history is the button-press log. SLS serializes the world state, not the log.

---

## 3. Problem Statement

SLS addresses five distinct but interconnected problems that arise from the current architecture of LLM inference systems:

**Session Ephemerality.** LLM sessions are stateless across session boundaries. When a session terminates — whether by user action, timeout, crash, or infrastructure event — the entire latent context is destroyed. The KV-cache, which may represent hundreds of thousands of tokens of accumulated computation, is deallocated. The nuanced, evolved "understanding" the model has built over the course of a long interaction vanishes. Replaying the conversation transcript to reconstruct context is not equivalent to continuing the original session: attention patterns differ on replay due to batching configuration, quantization precision, floating-point operation ordering, and RNG state divergence. The replayed session is a *similar* session, not the *same* session.

**Context Reconstruction Cost.** Even when transcript replay is acceptable as an approximation, it is computationally prohibitive for long sessions. Self-attention is O(n²) in sequence length; re-ingesting a 128K-token conversation history requires processing every token against every preceding token across all layers. For large models, this translates to minutes of GPU time and significant energy expenditure — computation that was already performed once and discarded. In latency-sensitive applications (real-time agents, interactive copilots, customer-facing systems), this reconstruction delay is unacceptable. The amortized cost of "just replay it" scales quadratically with session length, making it increasingly impractical as context windows grow.

**Forking Impossibility.** There is no mechanism in current inference systems to branch a session — to explore two different response strategies, two different tool calls, or two different reasoning paths from the same conversational state — without independently reconstructing the full upstream computation for each branch. This makes A/B testing, speculative execution, and exploratory dialogue prohibitively expensive. A fork operation should be O(1) with respect to session history (copy-on-write duplication of the state), not O(n²) (full replay for each branch).

**Cross-Environment Portability.** A session built on one machine, one GPU, or one data center cannot be migrated to another without full transcript replay. This prevents: live migration for load balancing (moving a session from a hot node to a cold one), failover (resuming a session on a backup node after hardware failure), user mobility (continuing a session started on one device from another), and multi-region serving (routing a session to the geographically nearest inference node mid-conversation). Current LLM infrastructure treats sessions as pinned to the hardware that created them.

**Auditability and Reproducibility.** Without a snapshot of the actual latent state at the moment of generation, there is no way to forensically reproduce the exact conditions under which a model produced a given output. Replaying the transcript with the same model version does not guarantee the same KV-cache state, and therefore does not guarantee the same output distribution. For regulated industries (finance, healthcare, legal), this means that the "reasoning" behind a model's output cannot be precisely reconstructed for audit. SLS provides the equivalent of a flight data recorder for LLM inference — a bit-exact snapshot of the computational state at any point of interest.

---

## 4. Prior Art and Technical Context

SLS does not emerge from a vacuum. Several existing systems and research efforts address adjacent problems. SLS synthesizes their insights and fills the gap between them.

### 4.1 State Stream Transformer (SST)

**Aviss, 2025.** The State Stream Transformer introduces a sliding-window latent state cache with weighted decay that maintains and evolves persistent latent processes throughout autoregressive generation. Unlike standard transformers, which discard feedforward network (FFN) intermediate states between tokens, SST preserves FFN state across generation steps, creating a continuous "state stream" alongside the standard token stream. This architectural modification alone — applied to frozen pretrained weights — yielded 89.01% accuracy on GSM-8K (0-shot) and 91.04% on ARC Challenge (0-shot CoT), demonstrating that persistent latent state enables fundamentally different reasoning strategies. SST validates the core premise of SLS: that the latent state carries information that cannot be reconstructed from the token sequence alone, and that preserving it has measurable computational value.

### 4.2 LLamaSharp State Persistence

**SciSharp / LLamaSharp.** LLamaSharp, a .NET binding for llama.cpp, provides a production-grade implementation of multi-level state serialization. It exposes two complementary persistence systems: *context state persistence*, which serializes the raw KV-cache and embeddings as an opaque binary blob via native llama.cpp APIs, and *executor state persistence*, which captures high-level conversation flow (token buffers, history, counters, sampling configuration). LLamaSharp demonstrates that KV-cache serialization is technically feasible today at the inference engine level — the question is not whether it can be done, but whether a standardized, portable format can be defined around it. LLamaSharp's serialization is engine-specific (llama.cpp only), format-opaque (no cross-engine compatibility), and unversioned. SLS proposes the standardization layer that sits above such engine-specific implementations.

### 4.3 DataStates-LLM

**Maurya et al., 2024–2026.** DataStates-LLM addresses the checkpointing problem for large-scale model *training*: how to efficiently serialize the distributed state of a trillion-parameter model across thousands of GPUs. Its key architectural contribution is the **State Provider** abstraction — composable entities that expose byte-range streaming interfaces for heterogeneous state fragments (GPU tensors, optimizer states, Python metadata). State Providers decouple state abstraction from data movement, enabling lazy, asynchronous snapshots that overlap with forward/backward passes. DataStates-LLM achieves up to 4× higher checkpointing throughput compared to prior solutions. While DataStates-LLM targets training checkpoints rather than inference state, its State Provider architecture and its treatment of heterogeneous shard aggregation directly inform the SLS payload design — particularly the challenge of serializing mixed-type data (quantized tensors, JSON metadata, binary adapter weights) into a single coherent artifact.

### 4.4 Agent Memory Patterns

**Phoenix, 2025.** James Phoenix's work on agent memory patterns defines a three-tier memory hierarchy for production AI agents: *Tier 1 — Session State* (in-memory during execution, lost when the session ends), *Tier 2 — File-Based Memory* (durable, human-readable files like TASKS.md that survive session boundaries), and *Tier 3 — Event-Sourced State* (append-only event logs enabling full history reconstruction and time-travel debugging). Phoenix identifies two critical failure modes: **context rot** (progressive degradation of context quality over extended sessions due to attention dilution) and the need for the **RALPH Loop** pattern (spawning fresh agents that inherit state from predecessors). The memory hierarchy is sound, but all three tiers operate at the *application* level — none captures the model's actual latent computational state. SLS introduces a Tier 0 beneath Phoenix's hierarchy: the raw inference state from which all higher-level representations are derived.

### 4.5 OpenAI Agents SDK Sessions

**OpenAI, 2025.** The OpenAI Agents SDK provides a Session object that manages multi-turn conversation history automatically, with built-in support for context trimming (removing oldest messages when approaching the context window limit) and context compression (summarizing prior conversation into a condensed form). Sessions eliminate manual message-list management and provide coherence across turns within a single runtime. However, Sessions operate entirely at the token level — they manage *which tokens are fed to the model*, not the model's internal state after processing those tokens. Trimming and compression are lossy operations that discard information the model had previously computed. SLS complements the Session abstraction by preserving the latent state that Sessions cannot: the KV-cache, attention patterns, and sampling state that exist only inside the inference engine.

> **Positioning SLS**
>
> SLS is the missing primitive that unifies these approaches. SST proves latent state persistence has value. LLamaSharp proves KV-cache serialization is feasible. DataStates-LLM provides the architectural patterns for heterogeneous state serialization. Phoenix's memory tiers define the application-level hierarchy that SLS undergirds. OpenAI's Sessions define the API surface that SLS enables beneath. SLS serializes the *actual computed state* — not a summary, not a reconstruction, not an approximation.

---

## 5. Technical Rationale

This section explains why SLS is the right abstraction — why serializing the latent state, rather than any higher-level representation, is the mechanistically correct approach.

### 5.1 KV-Cache Is the Ground Truth

The KV-cache is the authoritative representation of "what the model knows" at any point in a session. It contains the attention keys and values computed by every layer for every token in the context window. These values encode not just the semantic content of each token, but its *relationship to every other token* — contextual dependencies, coreference chains, discourse structure, and reasoning trajectories that emerge from multi-head attention across dozens or hundreds of layers.

Replaying the same token sequence through the same model does not guarantee an identical KV-cache. Three sources of divergence are well-documented:

- **Floating-point non-determinism.** GPU floating-point operations are not associative. Different execution orderings (due to thread scheduling, warp divergence, or kernel launch timing) produce numerically different results. Accumulated across billions of multiply-add operations over 128K tokens, these differences are measurable.
- **Quantization artifacts.** If the model is served at a different quantization level on replay (e.g., FP16 on the original run, INT8 on replay), the KV-cache values will differ systematically.
- **Batching effects.** Continuous batching systems (vLLM, TensorRT-LLM) interleave tokens from multiple requests. The order and composition of the batch affects memory layout and, in some implementations, the numerical results of fused attention kernels.

The KV-cache produced by a specific execution is therefore a unique artifact. It can be preserved, but it cannot be reliably recreated. This is the fundamental argument for serialization over replay.

### 5.2 Amortized Compute

A single serialize operation is O(1) relative to session length — it copies a fixed-layout memory region (the KV-cache) to a storage backend. The cost is proportional to the KV-cache size, which is determined by model architecture and context length, not by the computational history of the session.

Context reconstruction via replay is O(n²) in sequence length due to self-attention. For a 128K-token session on a large model, reconstruction can take minutes of GPU time. For a 1M-token session (as context windows continue to grow), it becomes impractical entirely.

| Operation | Time Complexity | 128K Tokens (est.) | 1M Tokens (est.) |
| --- | --- | --- | --- |
| serialize (SLS) | O(1) — memory copy | ~100ms–1s | ~1s–10s |
| deserialize (SLS) | O(1) — memory load | ~100ms–1s | ~1s–10s |
| Transcript replay | O(n²) — full self-attention | ~30s–5min | ~30min–hours |

SLS converts a variable, quadratically-scaling cost into a fixed, architecture-determined cost. This is the same class of optimization that makes memory-mapped files, process snapshots, and container image layers valuable — amortizing expensive computation into cheap persistence.

### 5.3 Composability

Once latent state is serialized into a file, it becomes a **first-class object** in the infrastructure stack. First-class objects can be:

- **Stored** — in object storage, on local disk, in a distributed cache.
- **Transmitted** — over a network, between data centers, across availability zones.
- **Versioned** — with content-addressable hashing, forming a DAG of session states.
- **Diffed** — comparing two SLS files to identify where sessions diverged.
- **Forked** — creating two independent sessions from one state via copy-on-write.
- **Merged** — potentially combining insights from two branches (an open research question).

This composability enables interaction paradigms that do not exist today: branching conversations, collaborative model interaction, checkpoint/rollback for agentic workflows, and version-controlled reasoning traces.

### 5.4 Separation of Concerns

SLS decouples *"what the model knows right now"* from *"how the model got there."* The conversation layer (managing messages, tool calls, user interactions) and the state management layer (persisting, restoring, migrating latent state) can evolve independently. A new conversation UI does not need to understand KV-cache internals. A new inference engine does not need to understand conversation semantics. SLS is the interface contract between them — analogous to how a filesystem interface decouples applications from storage hardware.

---

## 6. Minimal Specification (SLS v0.1)

This section defines a minimal, concrete specification sufficient for a reference implementation. The specification prioritizes simplicity and correctness over optimization. Future versions may introduce additional features (partial serialization, streaming, encryption-at-rest) as optional extensions.

### 6.1 File Format

- **Container:** A single binary file with extension `.sls`
- **Structure:** Header + Payload + Signature
- **Byte order:** Little-endian throughout

**Header (fixed-size, 256 bytes):**

| Field | Offset | Size | Type | Description |
| --- | --- | --- | --- | --- |
| Magic bytes | 0 | 4 | bytes | `SLS\x00` — identifies the file format |
| Spec version | 4 | 2 | uint16 | High byte = MAJOR, low byte = MINOR (e.g., 0x0001 = v0.1) |
| Architecture hash | 6 | 32 | bytes | SHA-256 of model architecture descriptor (layer count, hidden dim, head count, vocab size) |
| Weights hash | 38 | 32 | bytes | SHA-256 of the model weights file |
| Adapter hash | 70 | 32 | bytes | SHA-256 of active adapter weights; zeroed (0x00…) if no adapter active |
| Timestamp | 102 | 8 | uint64 | Capture time as Unix epoch in milliseconds |
| Compression codec | 110 | 1 | uint8 | 0x00 = none, 0x01 = zstd, 0x02 = lz4 |
| Payload length | 111 | 8 | uint64 | Total payload size in bytes (after compression, if applicable) |
| Metadata offset | 119 | 8 | uint64 | Byte offset from payload start to the JSON metadata block |
| Reserved | 127 | 129 | bytes | Zeroed; reserved for future header fields |

**Signature (appended after payload):** 64-byte Ed25519 signature over the concatenation of header + payload. Optional in v0.1; if absent, the 64 bytes are zeroed and a flag in the reserved header region indicates unsigned status.

### 6.2 Payload Sections

The payload is a sequence of typed sections. Each section is preceded by a 16-byte section header: 4-byte section type identifier, 4-byte reserved/flags, and 8-byte section length. Sections appear in the following canonical order:

| Section | Type ID | Contents | Encoding |
| --- | --- | --- | --- |
| kv_cache | 0x01 | Attention key and value tensors for all layers. Organized as `[layer][head][seq_pos][dim]` for keys and values separately. | Quantized tensor binary. dtype specified in section flags (FP16, BF16, FP32, INT8). Layout matches the source inference engine's memory format. |
| token_buffer | 0x02 | Full token ID sequence currently in the context window, in order. | uint32 array (supports vocab sizes up to 4B tokens). |
| sampling_state | 0x03 | RNG algorithm identifier, RNG internal state (seed + position), temperature, top_p, top_k, repetition penalty, frequency penalty, presence penalty. | JSON object. RNG state encoded as a hex string for portability. |
| system_bindings | 0x04 | SHA-256 hash of the system prompt, instruction-layer configuration, tool/function schemas, and any behavioral constraint descriptors. | JSON object containing hash references. Full system prompt text is **not** stored (to limit file size and avoid policy leakage); only the hash for verification. |
| adapter_state | 0x05 | Serialized LoRA or adapter delta weights active at capture time. Includes adapter rank, alpha, target modules, and the weight matrices themselves. | Safetensors-compatible binary format. Omitted entirely (zero-length section) if no adapter is active. |
| metadata | 0x06 | Human-readable provenance: creator identity, session ID (UUID), parent SLS hash (SHA-256, zeroed if root), tags (string array), free-form description, creation tool/version. | JSON object, UTF-8 encoded. |

### 6.3 Operations

The SLS runtime exposes four core operations. All operations are atomic — they either complete fully or leave no side effects.

| Operation | Signature | Semantics |
| --- | --- | --- |
| serialize | `serialize(session) → .sls file` | Captures the current inference state (KV-cache, token buffer, sampling state, system bindings, adapter state) and encodes it into an SLS binary file. The session continues unaffected after serialization. The operation is non-destructive — it is a snapshot, not a move. |
| deserialize | `deserialize(file) → session` | Validates the SLS file's compatibility (architecture hash, weights hash, spec version), decompresses the payload, and loads all sections into a new inference session. The session is immediately ready for generation — no replay or warm-up is required. Fails with a typed error if compatibility checks fail. |
| fork | `fork(file) → (session_a, session_b)` | Deserializes the SLS file into two independent sessions that share no mutable state. Implemented as deserialize + copy-on-write duplication of the KV-cache. Each branch can diverge independently from the same starting state. The fork point is recorded in both sessions' metadata as a shared parent hash. |
| diff | `diff(file_a, file_b) → delta` | Computes a structural diff between two SLS files. Reports: added/removed/modified tokens in the token buffer, divergence metrics for KV-cache values (L2 distance per layer), differences in sampling state, and metadata changes. Useful for debugging, auditing, and understanding where two session branches diverged. |

### 6.4 Compatibility Contract

An SLS file is loadable if and only if all of the following conditions are met:

1. **Architecture match.** The target inference engine's model architecture hash exactly matches the architecture hash in the SLS header. This ensures that the KV-cache tensor shapes, layer count, head count, and hidden dimensions are identical. A mismatch means the tensors cannot be loaded into the model's memory layout.

2. **Weights match or verified compatibility.** The target model's weights hash either (a) exactly matches the weights hash in the SLS header, or (b) a verified compatibility mapping exists in the runtime's compatibility registry declaring that the SLS weights hash is compatible with the target weights hash within acceptable fidelity bounds. Exact match guarantees bit-identical behavior. Compatibility mapping allows controlled degradation (e.g., loading state from a prior fine-tuning checkpoint into a newer one).

3. **Spec version support.** The SLS file's MAJOR version is supported by the runtime. The runtime MUST reject files with unsupported MAJOR versions. The runtime SHOULD process files with the same MAJOR version and a higher MINOR version by ignoring unknown sections and metadata fields.

---

## 7. Architecture Diagram Description

The following describes the SLS system architecture as a four-layer stack diagram. Each layer is a horizontal band spanning the full width of the diagram, with named components inside each band and directional arrows showing data flow between layers.

**Layer 1 — Application Layer (top)**

A single horizontal band labeled "Application Layer." Contains three representative components arranged side by side: *Chat UI*, *API Client*, and *Agent Framework*. Each component has a downward arrow labeled `serialize / deserialize / fork` pointing to Layer 2. This layer is the consumer of SLS — it issues commands but has no knowledge of the binary format or inference engine internals.

**Layer 2 — SLS Runtime (middle-upper)**

A wider horizontal band labeled "SLS Runtime." Contains four components arranged left to right:

- **State Capturer** — Positioned on the left. A downward arrow connects it to Layer 4 (Inference Engine) labeled "introspect: extract KV-cache, token buffer, sampling state." An upward arrow connects it to the Serializer labeled "raw state bundle."
- **Serializer / Deserializer** — Center-left. Accepts raw state bundles from the State Capturer. Performs compression (zstd/lz4), hashing (SHA-256 for all hash fields), and signature generation (Ed25519). A downward arrow connects to Layer 3 (Storage Backend) labeled `put(hash, bytes)`. A reverse upward arrow from Layer 3 is labeled `get(hash) → bytes`. The Deserializer path reverses the flow: reads from storage, validates signature, decompresses, and passes raw state to the State Capturer for injection into the Inference Engine.
- **Compatibility Checker** — Center-right. Receives the SLS header before deserialization begins. Compares architecture hash and weights hash against the target model. Queries an internal *Compatibility Registry* (depicted as a small database icon) for known-compatible weight mappings. Returns `PASS / FAIL / COMPATIBLE_WITH_FIDELITY_LOSS`.
- **Fork Manager** — Right side. Accepts a deserialized state bundle and performs copy-on-write duplication, producing two independent state bundles. Each is injected into a separate inference session via the State Capturer.

**Layer 3 — Storage Backend (middle-lower)**

A horizontal band labeled "Storage Backend." Contains a **pluggable interface** box on the left specifying the four operations: `put(hash, bytes)`, `get(hash) → bytes`, `list(prefix) → [hash]`, `delete(hash)`. To the right, three implementation boxes are shown side by side: *Local Filesystem*, *Object Storage (S3 / GCS / Azure Blob)*, and *Networked State Store (Redis / Distributed KV)*. Dashed lines connect each implementation to the interface box, indicating plug-in relationships.

**Layer 4 — Inference Engine (bottom)**

A horizontal band labeled "Inference Engine." Contains representative engines listed side by side: *vLLM*, *llama.cpp*, *TensorRT-LLM*, *ONNX Runtime*. Each engine has an upward-facing hook icon labeled "KV-cache extraction / injection hooks." Bidirectional arrows connect these hooks to the State Capturer in Layer 2. This is the layer that must be instrumented — each inference engine must expose APIs for reading and writing the KV-cache, token buffer, and sampling state.

**Data Flow Summary**

- **Serialize:** Application issues `serialize` → SLS Runtime's State Capturer pulls state from Inference Engine via extraction hooks → Serializer encodes, compresses, hashes, and signs → Storage Backend persists the `.sls` file.
- **Deserialize:** Application issues `deserialize` → Storage Backend retrieves the `.sls` file → Compatibility Checker validates hashes → Deserializer decompresses and decodes → State Capturer injects state into Inference Engine via injection hooks → session is live.
- **Fork:** Deserialize path executes once → Fork Manager duplicates the state bundle via copy-on-write → State Capturer injects each copy into a separate Inference Engine session → two independent sessions are live with identical starting state.

---

## 8. Versioning Strategy

### 8.1 File Versioning

Every SLS file contains a `parent_hash` field in its metadata section. This field holds the SHA-256 hash of the SLS file from which the current state was derived (or all zeros if the state is a root — captured from a fresh session with no SLS ancestry).

This creates a **Merkle-like directed acyclic graph (DAG)** of session states:

- A linear session (serialize at T1, continue, serialize at T2) produces a chain: T1 ← T2.
- A fork creates two children with the same parent hash: T1 ← T2a, T1 ← T2b.
- Multiple forks at different points create a tree. Multiple merges (if supported in future versions) create a DAG.

This DAG enables: **history traversal** (walking back through the parent chain to any ancestor state), **branch visualization** (rendering the tree of sessions derived from a common root), and **garbage collection** (identifying and reclaiming orphaned states that have no living descendants and no explicit retention policy).

### 8.2 Spec Versioning

The SLS specification uses **semantic versioning** encoded as MAJOR.MINOR in a single uint16:

| Change Type | Version Bump | Backward Compatibility | Examples |
| --- | --- | --- | --- |
| Breaking format changes | MAJOR | Old runtimes **cannot** read new files. New runtimes **may** read old files (at implementor's discretion). | Header layout change, payload encoding change, removal of a mandatory section. |
| Additive changes | MINOR | Old runtimes **can** read new files by ignoring unknown sections and metadata fields. | New optional metadata field, new compression codec, new optional payload section. |

A **spec version registry** (maintained as a publicly accessible document or API endpoint) maps each version number to its canonical format description, including header layout, section types, and supported compression codecs.

Runtime behavior requirements:

- Runtimes **MUST** reject SLS files with an unsupported MAJOR version with a clear error identifying the version mismatch.
- Runtimes **SHOULD** emit a warning when encountering unknown MINOR features (unrecognized section types, unknown metadata fields) and proceed with deserialization by skipping the unknown sections.
- Runtimes **MUST NOT** silently ignore a MAJOR version mismatch.

### 8.3 Model Compatibility Evolution

When a model is fine-tuned, updated, or re-quantized, a new weights hash is generated. This would normally invalidate all existing SLS files for that model. To support controlled migration, the specification defines a **compatibility mapping** mechanism:

- A compatibility mapping is a signed declaration (maintained per-organization or per-model-family) that states: "SLS files created with weights hash A are loadable into a model with weights hash B with quantified fidelity loss."
- **Fidelity loss** is measured as the KL-divergence between the output probability distributions produced by: (1) the original model with the original KV-cache, and (2) the target model with the loaded KV-cache, evaluated over a standardized reference prompt set.
- The Compatibility Checker in the SLS Runtime consults these mappings during deserialization. If a mapping exists and the fidelity loss is within the runtime's configured threshold, deserialization proceeds with a logged warning. If no mapping exists or the fidelity loss exceeds the threshold, deserialization fails.

---

## 9. Security Considerations

SLS files are high-value artifacts. They contain the complete computational context of a session, which may include sensitive data, proprietary instructions, and information about model internals. The following threat vectors must be addressed by any production implementation.

**State Exfiltration.** An SLS file contains the full context of a session — every token processed, every attention pattern computed, every piece of data discussed. If a session involved sensitive personal data, trade secrets, medical records, or classified information, that data is embedded in the KV-cache and token buffer in its entirety. *Mitigations:* SLS files must be encrypted at rest using AES-256-GCM (or equivalent AEAD cipher). In transit, SLS files must be transported over TLS 1.3 or later. Access control on SLS files must be at least as restrictive as the data classification of the most sensitive content discussed in the session. SLS files should inherit the access policies of the session that created them.

**State Injection / Poisoning.** A maliciously crafted SLS file could inject adversarial KV-cache states designed to manipulate model behavior upon deserialization. Potential attacks include: bypassing safety alignment by injecting KV-cache values that simulate a jailbroken context, inducing specific outputs by crafting attention patterns that bias generation toward attacker-chosen text, or degrading model performance by injecting noise into the KV-cache. *Mitigations:* Cryptographic signatures (Ed25519) on all SLS files, verified before deserialization. Provenance chains via parent hashes — runtimes can enforce that only SLS files with a trusted provenance chain (rooted in a known-good state) are loadable. Optional runtime validation: after deserialization, run a canary prompt set and verify that the model's outputs fall within expected distributions.

**Model Extraction.** The KV-cache leaks information about model weights. Attention key and value vectors are linear projections of the model's hidden states, which are themselves functions of the model's weight matrices. An adversary with access to many SLS files from the same model — particularly files created from diverse prompts — could potentially reconstruct approximations of the model's weight matrices via linear algebra. *Mitigations:* Differential privacy noise injection on serialized KV-cache values (calibrated to preserve output distribution while obscuring weight information). Access rate limiting — enforce quotas on the number of SLS files any single entity can create or retrieve. Comprehensive audit logging of all serialize/deserialize operations.

**Replay Attacks.** An attacker replays an old SLS file to restore a session to a state that predates a policy update, safety patch, or system prompt change. This could resurrect a jailbroken context, bypass newly deployed safety filters, or restore a session to a state where the model had been manipulated. *Mitigations:* SLS files carry a timestamp. Runtimes can enforce a maximum SLS age — files older than the threshold are rejected. SLS files can be bound to a policy version (stored in the system bindings section) — runtimes can enforce a minimum policy version and reject files created under outdated policies. Revocation lists can explicitly invalidate specific SLS file hashes.

**Integrity.** SLS files must be tamper-evident. Any modification to the header, payload, or any individual section must be detectable. *Mitigations:* The Ed25519 signature in the file covers the entire concatenation of header + payload. Any bit-level modification invalidates the signature. Runtimes MUST verify the signature before deserialization (when signatures are present). The SHA-256 hashes in the header (architecture, weights, adapter) provide additional integrity checks — if an attacker modifies the payload but not the header, the hash mismatches will be detected during section loading.

> **Security Note**
>
> SLS files should be treated with the same security posture as database backups or memory dumps. They are not "just files" — they are complete computational state snapshots that may contain any data the model processed. Organizations deploying SLS must integrate SLS file management into their existing data governance, classification, and retention frameworks.

---

## 10. Example Use Cases

### 10.1 Long-Running Research Sessions

A researcher engaged in a multi-day analysis of a complex dataset uses a model with a 128K-token context window. Over the course of the collaboration, the model accumulates deep context about the dataset's structure, the researcher's hypotheses, prior analysis results, and domain-specific terminology. With SLS, the researcher saves state at key milestones — after initial data exploration, after hypothesis formation, after each analytical pass. If a line of reasoning reaches a dead end, the researcher loads a prior state and branches in a different direction without re-ingesting the entire conversation. The KV-cache preserves the model's nuanced understanding of the data at each milestone, including attention patterns that would be lost on transcript replay.

### 10.2 Customer Support Handoff

An AI agent handles a complex, multi-step support case involving account history, troubleshooting steps already attempted, and escalation context. When the support shift ends, or when the case requires handoff to a specialist, the agent serializes its state. The receiving agent (or a human reviewer inspecting the case) deserializes into the exact same computational context — the same understanding of the customer's problem, the same awareness of what has been tried, the same conversational nuance. Zero context loss, zero reconstruction cost, zero risk of the new agent contradicting prior commitments.

### 10.3 Agentic Checkpoint/Resume

An autonomous agent executing a multi-step plan (code generation → testing → deployment → monitoring) serializes its state before each risky action. If a deployment fails or a test produces unexpected results, the agent rolls back to the last known-good state and retries with a different strategy. This is checkpoint/resume for AI agents — the same pattern that makes database transactions and version control systems reliable, applied to model inference. Without SLS, a failed step requires restarting the entire agent loop from the beginning, replaying all prior reasoning.

### 10.4 A/B Testing Conversational Strategies

A product team wants to compare two response strategies for a customer-facing model: Strategy A (concise, direct answers) vs. Strategy B (detailed, explanatory answers). With SLS, they fork the session at the decision point, run both strategies from an identical starting state, and compare outcomes (user satisfaction, task completion, follow-up questions). Without SLS, achieving identical starting states requires either exact transcript replay (which is not guaranteed to produce identical latent state) or running both strategies from the very beginning of the conversation (doubling the compute cost).

### 10.5 Regulatory Compliance and Audit

A financial advisory AI system serializes state at every decision point — before each investment recommendation, risk assessment, or compliance determination. When regulators audit the system, they can deserialize any SLS file and inspect the exact computational state that produced a given output. They can verify the model version, adapter configuration, system prompt, and KV-cache state. They can run additional prompts against the deserialized state to understand how the model would have responded to alternative questions at that moment. This provides a forensic trail with a fidelity that transcript logs cannot match.

### 10.6 Load Balancing and Migration

A cloud inference provider runs sessions across a fleet of GPU nodes. When a node approaches capacity, the provider serializes active sessions on that node, transmits the SLS files to underutilized nodes, and deserializes them — achieving true stateful live migration with no transcript replay. Failover becomes seamless: if a node fails, sessions can be restored on any compatible node from the most recent SLS snapshot. This is the LLM equivalent of virtual machine live migration — a capability that transformed cloud computing and does not yet exist for model inference.

### 10.7 Collaborative Model Interaction

Multiple team members contribute to a shared analytical session. The SLS file serves as the "shared document." A team member works on a line of analysis, serializes the state, and shares the SLS file. A colleague deserializes, reviews the model's current understanding, forks to explore a different angle, and serializes their branch. The team can visualize the branching tree of session states, compare branches, and decide which to continue — a version-controlled, branching workflow for model interaction, analogous to Git for code.

---

## 11. Open Questions and Future Work

This initial proposal leaves several significant questions unresolved. These are flagged as explicit open problems for the community to investigate.

**Cross-quantization compatibility.** How should SLS handle state captured at one quantization level (e.g., FP16 KV-cache) being loaded into a model running at a different quantization level (e.g., INT4 weights with FP16 KV-cache, or INT8 KV-cache)? Is lossless conversion possible? If not, can fidelity loss be bounded and quantified? Does the quantization of KV-cache values affect output distribution differently than quantization of weights?

**SLS file size at scale.** For a 405B-parameter model (e.g., Llama 3.1 405B) with a 128K-token context window, the KV-cache alone can exceed 100 GB. For future models with 1M+ token context windows, SLS files could reach terabytes. What is the practical upper bound on file size for serialization, transmission, and storage? Are there compression strategies specific to KV-cache data (exploiting the structure of attention tensors) that could reduce file size by an order of magnitude?

**State-level merging.** Can two SLS files — representing two branches forked from a common ancestor — be meaningfully merged? Token buffers can be diffed and merged analogously to text files, but KV-cache tensors encode non-linear, high-dimensional relationships. Is there a principled way to combine two KV-cache states, or is merging fundamentally intractable at the latent level? Would re-processing the merged token buffer through the model be the only viable approach?

**Multi-party data governance.** If an SLS file captures a session involving data from multiple parties (e.g., a multi-user conversation, or a session that ingested documents from different data owners), whose access policies govern the file? How are data sovereignty and right-to-deletion requirements (e.g., GDPR) enforced when the data is embedded in KV-cache tensors rather than stored as discrete records?

**Mixture-of-Experts (MoE) architectures.** In MoE models, only a subset of expert modules are activated for any given token. The KV-cache structure may differ from dense models, and the routing decisions (which experts were activated for which tokens) constitute additional state that affects model behavior. Should SLS capture expert routing state? How does partial expert activation affect KV-cache compatibility across sessions?

**Partial serialization.** For bandwidth-constrained or latency-sensitive scenarios, should the spec support serializing a subset of the state — for example, only the KV-cache for the last N layers (since lower layers tend to encode more generic, model-level features), or only the KV-cache for the most recent K tokens? What is the fidelity-size tradeoff curve for partial serialization, and can it be characterized analytically?

---

## 12. Conclusion

Serializable Latent State is a foundational infrastructure primitive for the LLM era — the equivalent of virtual memory pages for operating systems, container images for deployment, or write-ahead logs for databases. It transforms the model's ephemeral computational context into a durable, portable, versionable, forkable artifact that can be managed with the same rigor and tooling that the industry applies to every other form of critical state. This document is an initial proposal — a v0.1 specification and architectural framework — intended to be implemented, stress-tested, and refined by the community. The core thesis is simple and mechanistically grounded: the KV-cache is the ground truth of what the model "knows," and we should be able to save it, load it, and build on it.
