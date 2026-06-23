# SLS Implementation Guide
*Building LLM Save Files — From Proof of Concept to Production*

**Companion to the SLS Proposal Document**

| Field | Value |
| --- | --- |
| **Version** | v0.1 |
| **Date** | April 2026 |
| **Audience** | Systems Engineers, ML Infrastructure Developers, Open-Source Contributors |

---

## 1. Executive Summary

This document is the engineering blueprint for the Serializable Latent State (SLS) system. It does not re-argue the case for LLM save files — that is covered in the companion proposal. Instead, it specifies how to build the system by unifying four existing anchors in the inference ecosystem:

- **llama.cpp** — `llama_state_get_data` / `llama_state_set_data` functions and server slot save/restore API
- **HuggingFace Transformers** — `DynamicCache` / `StaticCache` classes backed by torch tensors with no native disk persistence
- **vLLM** — paged block-based KV cache with hash-indexed prefix caching but no disk serialization
- **DataStates-LLM** — CUDA-aware async checkpointing with `io_uring` delivering 3.9× write throughput over TorchSnapshot

The implementation strategy is to define a portable binary container format (`.sls`) and build runtime libraries that bridge these systems. Rust provides the core format library for safety and performance. PyO3 bindings expose a Python-first API surface for the HuggingFace and vLLM ecosystems. The work is phased: prove deterministic round-trip (Phase 0), define the container format and CLI (Phase 1), integrate with HuggingFace Transformers (Phase 2), extend inference servers (Phase 3), and harden for production with encryption, versioning, and cloud storage (Phase 4).

---

## 2. Implementation Anchors — What Already Exists

The SLS system is not built from scratch. Four existing codebases provide the foundation. The table below summarizes what each provides and what is missing — the gaps define the SLS implementation scope.

| System | What Exists | What's Missing |
| --- | --- | --- |
| **llama.cpp** | Full context state serialization, server slot save/restore via REST API, binary slot files, near-instant restore, deterministic round-trip verification example | No standardized container format, no metadata envelope, no versioning, no integrity checking, no cross-model compatibility validation |
| **HuggingFace Transformers** | `DynamicCache`, `StaticCache`, `QuantizedCache`, `OffloadedCache` classes; `past_key_values` pipeline; torch tensor serialization; sliding window and chunked attention cache support | No native disk persistence API, no structured serialization format, no sampler state capture, no model compatibility binding |
| **vLLM** | Paged block-based KV cache, `KVCacheManager` with hash-based O(1) prefix lookup, `BlockPool` for memory management, block sharing across requests via prefix caching | No disk serialization — entirely GPU-memory-resident, no state export/import mechanism |
| **DataStates-LLM** | CUDA-aware async checkpointing, `io_uring`-based file I/O, State Providers for composable byte-range streaming, 3.9× write throughput over TorchSnapshot, 7.6× over standard approaches, I/O coalescing and aggregation strategies | Training-focused (not inference-state-oriented), no portable container format, no inference-specific metadata |

### 2.1 llama.cpp — State Serialization Primitives

llama.cpp provides the most complete existing implementation of inference state serialization. The core API is two functions:

- `llama_state_get_data(ctx, dst, size)` — serializes full context state (KV cache + internal bookkeeping) to a byte buffer
- `llama_state_set_data(ctx, src, size)` — restores context state from a byte buffer

The server exposes this via REST endpoints: `/slots/<id>/save` and `/slots/<id>/restore`. Binary slot files are hundreds of MB for long contexts but restore near-instantly compared to re-processing the full prompt. The `save-load-state` example demonstrates deterministic round-trip: generate tokens, save state, load into a fresh context, continue generation, and verify output matches.

> **Gap:** The serialized bytes are an opaque blob. There is no header identifying which model produced the state, no version field, no checksum, and no mechanism to detect if a state file is loaded into an incompatible model. SLS wraps this blob in a structured container.

### 2.2 HuggingFace Transformers — Cache Class Hierarchy

The Transformers library defines an extensible cache system rooted in the `Cache` base class. Subclasses include:

- `DynamicCache` — grows dynamically as tokens are processed, stores per-layer key/value tensors
- `StaticCache` — pre-allocated fixed-size cache for compile-friendly workloads
- `QuantizedCache` — stores KV pairs in quantized format to reduce memory
- `OffloadedCache` — offloads inactive layers to CPU to fit longer contexts in GPU memory

All caches expose `past_key_values` as a tuple of torch tensors and can be passed to `model.generate()`. Serialization is possible via `torch.save()` / `torch.load()`, but there is no built-in API to save a cache to disk with metadata, no sampler state capture alongside the cache, and no validation that a saved cache matches the current model.

### 2.3 vLLM — Paged KV Cache Architecture

vLLM's KV cache is managed as a pool of fixed-size blocks, analogous to virtual memory pages. The `KVCacheManager` tracks which blocks are allocated to which sequences, and a hash table enables O(1) prefix lookup — if two requests share a prompt prefix, they share KV cache blocks. This design is optimized for throughput in continuous batching scenarios.

> **Gap:** The entire system is GPU-memory-resident. There is no mechanism to export block contents to disk or import them. SLS must interface with the block allocator to walk allocated blocks, serialize their contents, and reconstruct the page table on restore.

### 2.4 DataStates-LLM — High-Performance I/O Patterns

DataStates-LLM focuses on training checkpoint performance but provides I/O patterns directly applicable to SLS. Its key contribution is CUDA-aware async checkpointing: tensors are streamed from GPU memory to NVMe storage using `io_uring` for non-blocking file I/O, achieving 3.9× throughput improvement over TorchSnapshot. The State Provider abstraction enables composable byte-range streaming, and I/O coalescing reduces system call overhead.

> **Gap:** DataStates-LLM targets training checkpoints (full model weights + optimizer state), not inference state (KV cache + sampler). SLS adopts its I/O strategies — specifically `io_uring` and streaming writes — but applies them to a different payload with different metadata requirements.

---

## 3. Technology Stack Decisions

Each decision below is presented as: **Decision → Rationale → Alternatives Considered → Risk.**

### 3.1 Core Library Language: Rust

| Aspect | Detail |
| --- | --- |
| **Decision** | Implement `sls-core` (format reader/writer, integrity engine, CLI) in Rust. |
| **Rationale** | Memory safety guarantees are critical for a library that handles memory-mapped state files, including potentially untrusted ones. Zero-cost FFI to C (for llama.cpp integration) and Python (via PyO3). Performance parity with C++ for I/O-bound workloads. Strong type system reduces format-level bugs. |
| **Alternatives** | C++ — rejected due to memory safety burden for a format that handles untrusted state files. Python — rejected for performance on multi-GB I/O. C — rejected due to lack of modern type system and ecosystem tooling. |
| **Risk** | Smaller ML ecosystem contributor pool familiar with Rust. Mitigated by Python-first API surface — Rust internals are an implementation detail for most users. |

### 3.2 Python Bindings: PyO3

| Aspect | Detail |
| --- | --- |
| **Decision** | Use PyO3 to expose `SLSFile`, `SLSReader`, `SLSWriter` classes to Python. |
| **Rationale** | First-class Python integration is non-negotiable for the HuggingFace and vLLM ecosystems. PyO3 provides ergonomic Rust-to-Python bindings with automatic type conversion, exception handling, and GIL management. |
| **Alternatives** | ctypes/cffi — rejected as too fragile for complex types (nested structs, byte buffers, error variants). pybind11 — would require a C++ shim layer. |
| **Risk** | PyO3 version churn and compatibility with Python minor releases. Mitigate by pinning to PyO3 stable and testing across Python 3.10–3.12. |

### 3.3 Serialization Format: Custom Binary with Memory-Mapped Sections

| Aspect | Detail |
| --- | --- |
| **Decision** | Define a custom binary container. Not protobuf, not flatbuffers, not SafeTensors. KV cache tensors are multi-GB — the format must support zero-copy mmap access, not deserialization. Fixed 256-byte header, section table with offset/length/checksum per payload, raw tensor data aligned to page boundaries. |
| **Rationale** | Existing serialization formats either lack metadata envelopes (SafeTensors), don't support non-tensor sections (SafeTensors), or add deserialization overhead on multi-GB payloads (protobuf, flatbuffers). The SLS format is purpose-built for inference state: mixed payload types (tensors + RNG state + JSON metadata), page-aligned for mmap, and checksummed per-section for streaming verification. |
| **Alternatives** | SafeTensors — considered seriously. Good for weights but lacks metadata envelope, non-tensor sections, and per-section integrity checking. Could be used as the tensor payload format within SLS sections in a future iteration. |
| **Risk** | Custom format maintenance burden. A new format requires its own specification, compliance tests, and tooling. Mitigate by keeping the format simple and publishing a detailed specification from Phase 1. |

### 3.4 Integrity: BLAKE3

| Aspect | Detail |
| --- | --- |
| **Decision** | BLAKE3 for all checksums: per-section checksums and whole-file root hash. |
| **Rationale** | Parallelizable by design — critical for multi-GB payloads. Approximately 4× faster than SHA-256 on modern hardware. Tree-hashing mode enables streaming verification without buffering entire sections. 256-bit output matches SHA-256 security level. |
| **Alternatives** | SHA-256 — rejected as too slow for real-time save on large states (would add seconds to save latency on 70B model states). CRC32 — rejected as insufficient for integrity guarantees on security-sensitive state files. |
| **Risk** | BLAKE3 is less universally supported than SHA-256. Some enterprise environments may require SHA-256 for compliance. Mitigate by making the hash algorithm a format field — default BLAKE3, optional SHA-256 fallback. |

### 3.5 Compression: zstd (Optional, Per-Section)

| Aspect | Detail |
| --- | --- |
| **Decision** | Optional zstd compression on a per-section basis. Sections declare compressed/uncompressed in their descriptors. |
| **Rationale** | Achieves 1.3–1.8× compression ratio on float16/bfloat16 tensor data. Dictionary training possible for repeated structural patterns across saves. Streaming compression enables async I/O overlap. Per-section granularity allows compressing metadata (small, high ratio) while leaving tensor data uncompressed (large, low ratio, mmap-compatible). |
| **Alternatives** | lz4 — faster but worse compression ratio. No compression — simpler but storage-heavy, especially for cloud backends where bandwidth costs money. |
| **Risk** | Compression adds latency to the save path. Mitigate by making compression optional and defaulting to off for latency-sensitive workloads. Compressed sections cannot be mmap'd — must be decompressed to a buffer. |

### 3.6 Storage Backends: Pluggable Trait

| Aspect | Detail |
| --- | --- |
| **Decision** | Define a `StorageBackend` trait with implementations: `LocalFS` (default, mmap-capable), `S3Compatible` (for cloud), `InMemory` (for testing). |
| **Rationale** | Production inference runs in cloud environments where S3-compatible object storage is the primary persistence layer. Local filesystem is sufficient for development and edge deployment. The trait abstraction enables testing without disk I/O. |
| **Alternatives** | Filesystem-only — rejected because cloud deployment is the primary production target. Requiring filesystem for cloud would force operators to manage EBS volumes or NFS mounts. |
| **Risk** | Backend abstraction adds complexity. Object storage semantics (eventual consistency, no append) differ from filesystem semantics (POSIX, mmap). Mitigate by designing the trait around the lowest-common-denominator operations: read, write, list, delete. |

---

## 4. Phased Implementation Roadmap

### 4.1 Phase 0: Proof of Concept — "Can We Round-Trip?" (Weeks 1–3)

**Goal:** Prove deterministic state round-trip on a real model.

**Tasks:**

1. Fork llama.cpp's `examples/save-load-state/save-load-state.cpp`
2. Wrap `llama_state_get_data` output in a minimal header:
   - Magic bytes: `SLS\x00`
   - Version: `u16`
   - Model architecture hash: BLAKE3 of GGUF metadata (32 bytes)
   - Token count: `u32`
   - Timestamp: `u64`
3. Build a Python script that:
   - Generates N tokens from a prompt
   - Saves state with header
   - Loads state into a fresh context
   - Generates N more tokens
   - Asserts output matches original continuation
4. Measure save/load times across model sizes and context lengths

**Deliverables:**
- Working round-trip demo script
- Measured save/load times for 7B and 70B models at 4K and 32K context lengths

**Success Criteria:**

| Criterion | Target |
| --- | --- |
| Output determinism | Bit-identical output after state restoration |
| Save latency (7B / 4K context) | < 2 seconds on consumer NVMe |
| Model coverage | LLaMA 3 8B confirmed working |

### 4.2 Phase 1: The .sls Container Format + CLI (Weeks 4–8)

**Goal:** Define and implement the portable binary container format.

**Binary Format Specification**

The `.sls` file is composed of four regions: a fixed header, a section table, page-aligned payload sections, and a trailing root hash.

**Header (256 bytes, fixed):**

| Offset | Size | Field | Type |
| --- | --- | --- | --- |
| 0x00 | 4 B | Magic bytes | `SLS\x00` |
| 0x04 | 2 B | Format version major | `u16` |
| 0x06 | 2 B | Format version minor | `u16` |
| 0x08 | 32 B | Model architecture hash | BLAKE3 |
| 0x28 | 16 B | Model quantization identifier | bytes |
| 0x38 | 4 B | Context length at save | `u32` |
| 0x3C | 4 B | Token count | `u32` |
| 0x40 | 8 B | Creation timestamp | `u64` (Unix epoch) |
| 0x48 | 32 B | Parent state hash (for forking) | BLAKE3 (zeroed if root) |
| 0x68 | 4 B | Flags bitfield | `u32` (compressed, encrypted, partial) |
| 0x6C | 4 B | Section count | `u32` |
| 0x70 | 144 B | Reserved padding | zeroed |

**Section Table:** Array of section descriptors, immediately following the header.

| Field | Size | Type |
| --- | --- | --- |
| Section type | 2 B | `u16` enum |
| Flags | 2 B | `u16` |
| Offset | 8 B | `u64` |
| Length (on disk) | 8 B | `u64` |
| Uncompressed length | 8 B | `u64` |
| Checksum | 32 B | BLAKE3 |

**Section Type Enum:**

| Value | Name | Description |
| --- | --- | --- |
| 0x01 | `KV_CACHE` | Key-value cache tensors (per-layer) |
| 0x02 | `TOKEN_BUFFER` | Token IDs processed so far |
| 0x03 | `SAMPLER_STATE` | RNG state + sampling parameters |
| 0x04 | `SYSTEM_BINDINGS` | Model hash, context configuration |
| 0x05 | `ADAPTER_WEIGHTS` | LoRA adapter state (optional) |
| 0x06 | `METADATA_JSON` | Arbitrary JSON metadata |

**Payload sections:** Page-aligned (4096-byte boundary) raw data. **Trailing root hash:** BLAKE3 of header + section table (32 bytes at end of file).

**CLI Commands:**

| Command | Description |
| --- | --- |
| `sls save --model <gguf> --state <binary> --output <file.sls>` | Wraps raw state blob in `.sls` container with header, integrity, and metadata |
| `sls load --file <file.sls> --verify` | Validates integrity (root hash + per-section checksums), extracts state |
| `sls inspect <file.sls>` | Prints header, section table, model hash, token count, timestamps in human-readable format |
| `sls fork <file.sls> --output <forked.sls>` | Creates new state file with parent hash set to source file's root hash |
| `sls diff <a.sls> <b.sls>` | Compares headers and section checksums, reports divergence points |

**Deliverables:**
- Rust crate: `sls-core`
- Rust binary: `sls-cli`
- Format specification: `spec/FORMAT.md` v0.1

### 4.3 Phase 2: HuggingFace Transformers Integration (Weeks 9–14)

**Goal:** Make SLS a first-class citizen in the HuggingFace generation pipeline.

**SLSCache Class**

Implement `SLSCache` extending HuggingFace's `Cache` base class:

| Method | Source | Behavior |
| --- | --- | --- |
| `update()` | Inherited | Standard KV cache update per generation step |
| `get_seq_length()` | Inherited | Returns current sequence length from cache |
| `get_max_cache_shape()` | Inherited | Returns maximum cache dimensions |
| `reset()` | Inherited | Clears cache state |
| `serialize(path)` | **New** | Captures all layer KV tensors + sampler RNG state + token buffer into `.sls` container |
| `deserialize(path)` | **New** | Restores cache from `.sls` file, validates model compatibility via architecture hash |
| `fork()` | **New** | Deep copy with new parent hash for branching conversations |

**Sampler State Capture**

The `SAMPLER_STATE` section stores a fixed-size parameter struct followed by variable-length RNG bytes:

- **Parameters captured:** `temperature`, `top_p`, `top_k`, `repetition_penalty`
- **RNG state:** `torch.get_rng_state()` + CUDA RNG state if applicable
- **Format:** Fixed-size struct (parameter values as `f32`/`u32`) + variable-length RNG bytes

**Model Compatibility Checker**

Architecture hash computation:

```
hash = BLAKE3(model_type + hidden_size + num_layers + num_heads + vocab_size + quantization_config)
```

On load: compare saved hash vs. current model hash. Fail with descriptive error on mismatch, including which fields differ.

**Integration with generate()**

Two new parameters on `model.generate()`:

- `save_state_path: Optional[str]` — automatically saves state after generation completes
- `load_state_path: Optional[str]` — loads state before generation starts, skipping prefix computation

**Deliverables:**
- Python package: `pip install sls-transformers`
- HuggingFace integration with all Cache subclasses
- Example Jupyter notebooks

### 4.4 Phase 3: Inference Server Integration (Weeks 15–22)

**Goal:** Add SLS support to production inference servers.

**llama.cpp Server Extension**

Extend existing slot save/restore to use `.sls` format. Add new REST endpoints:

| Endpoint | Method | Description |
| --- | --- | --- |
| `/v1/states/save` | POST | Save current state to `.sls` file, return state ID and metadata |
| `/v1/states/load` | POST | Load state from `.sls` file by ID or path |
| `/v1/states/fork` | POST | Fork existing state, return new state ID |
| `/v1/states/{id}` | GET | Inspect state metadata |
| `/v1/states` | GET | List all saved states |

Backward compatibility: detect raw binary vs. `.sls` format on restore by checking magic bytes.

**vLLM Plugin**

Implement `SLSBlockExporter` that interfaces with `KVCacheManager`:

- **Save path:** Walk allocated blocks, serialize page table + block contents to SLS `KV_CACHE` section. Export hash table entries alongside blocks for cache-warm restore.
- **Load path:** Allocate blocks via `BlockPool`, populate from SLS payload, reconstruct page table.
- **Async I/O:** Use `io_uring` (via `tokio-uring` in Rust) for non-blocking state writes. Overlap serialization with ongoing inference — save does not block generation.

> **Performance Target**
>
> Save latency < 100ms for a 7B model at 8K context. Async I/O path must not degrade inference throughput by more than 1%.

REST API additions to vLLM's OpenAI-compatible server follow the same `/v1/states/*` endpoint pattern as the llama.cpp extension for API consistency.

### 4.5 Phase 4: Advanced Features (Weeks 23–32)

**Goal:** Production hardening and advanced capabilities.

**Merkle DAG Versioning**
- Parent hash in header forms a hash-linked chain (analogous to git commits)
- `sls log <file.sls>` — walks parent chain, displays fork history
- `sls merge <a.sls> <b.sls>` — experimental: merge divergent states (research-grade, not production)
- Content-addressable store keyed by root hash (like git objects)

**Encryption at Rest**
- AES-256-GCM per section, key derived from user-provided passphrase via Argon2id
- Encrypted sections have `ENCRYPTED` flag in section descriptor
- Header and section table remain plaintext — metadata is non-sensitive by design
- Key rotation: re-encrypt with new key without deserializing KV cache (stream cipher re-wrap)

**Cross-Quantization Compatibility Layer**
- Experimental: dequantize KV cache to FP16 reference, re-quantize to target format
- Fidelity metric: measure KL-divergence of output distributions before/after conversion
- Warning threshold: KL-div > 0.01 nats → warn user of potential quality degradation

**Cloud Storage Backends**
- **S3-compatible:** Multipart upload for large states, streaming download with range requests
- **GCS:** Resumable uploads with similar pattern
- **State registry:** JSON index file listing all states with metadata, searchable by model/timestamp/token count

**Partial Serialization**
- Save only specific layers' KV cache (e.g., last N layers for sliding window models)
- Selective section save: skip ADAPTER_WEIGHTS if not using LoRA
- Delta saves: serialize only blocks that changed since last save (dirty-bit tracking in block allocator)

## 5. Architecture Diagram Descriptions

The following tables describe the system architecture as layered component diagrams. These are intended to be rendered as visual diagrams in engineering documentation tools.

### 5.1 Component Architecture (Four-Layer Stack)

| Layer | Components | Responsibility |
| --- | --- | --- |
| **Application Layer** (top) | User code, `generate()` calls, CLI commands, REST API clients | Initiates save/load/fork/diff operations |
| **SLS Runtime** | `SLSFile` (read/write/fork/diff), `SLSCache` (HuggingFace bridge), `ModelCompatibility` (hash validation), `SamplerCapture` (RNG + params) | Core logic: format handling, model binding, state composition |
| **Storage Layer** | `StorageBackend` trait → `LocalFS` / `S3Compatible` / `InMemory`, `IntegrityEngine` (BLAKE3), `CompressionEngine` (zstd) | Persistence, integrity verification, compression |
| **Inference Engine Layer** (bottom) | llama.cpp context (`llama_state_get/set_data`), HuggingFace `past_key_values`, vLLM `KVCacheManager` / `BlockPool` | Source and destination of raw inference state |

### 5.2 Save Flow (Data Path)

| Step | Operation | Data |
| --- | --- | --- |
| 1 | Application calls `sls.save(model, state, path)` | Model reference, context handle, output path |
| 2 | SLS Runtime extracts components from inference engine | KV cache tensors, token buffer, sampler state (RNG + params), system bindings (model hash, context config) |
| 3 | Each component serialized into section payload | Raw bytes per section |
| 4 | Sections optionally compressed (zstd) and checksummed (BLAKE3) | Compressed bytes + 32-byte checksums |
| 5 | Header assembled | Magic, version, model hash, token count, parent hash, flags |
| 6 | Section table written with offsets/lengths/checksums | Section descriptors array |
| 7 | Payloads written page-aligned | Raw (or compressed) section data, padded to 4096-byte boundaries |
| 8 | Root hash computed and appended | `BLAKE3(header + section table)` → 32 bytes |
| 9 | File written to storage backend | Sync write or async via `io_uring` |

### 5.3 Load Flow (Data Path)

| Step | Operation | Validation |
| --- | --- | --- |
| 1 | Application calls `sls.load(path, model)` | — |
| 2 | Storage backend reads header (256 bytes) | — |
| 3 | Parse header fields | Magic bytes == `SLS\x00`, format version supported |
| 4 | Model compatibility check | Saved architecture hash == current model architecture hash |
| 5 | Root hash verification | Recomputed `BLAKE3(header + section table)` == trailing root hash |
| 6 | Section table parsed, per-section checksums verified | Each section checksum matches payload |
| 7 | KV cache section mmap'd or loaded into GPU memory | — |
| 8 | Token buffer restored to context | Token count matches header field |
| 9 | Sampler state restored (RNG + parameters) | — |
| 10 | Cache returned to inference engine | As `past_key_values` or loaded into `KVCacheManager` |

---

## 6. Testing Strategy

### 6.1 Test Levels

| Level | Name | Description | Scope |
| --- | --- | --- | --- |
| 1 | **Determinism** | Generate → Save → Load → Generate. Assert bit-identical output after restoration. | Model families: LLaMA, Mistral, Qwen. Quantizations: Q4_K_M, Q8_0, FP16. Context lengths: 1K, 4K, 16K, 32K. |
| 2 | **Integrity** | Corrupt individual bytes in header, section table, and payload. Assert load fails with specific error codes. | Truncated files, wrong magic bytes, version mismatches, zeroed checksums, flipped bits in payload. |
| 3 | **Compatibility** | Save with model A, attempt load with model B (different architecture). Assert rejection with descriptive error. | Cross-architecture (LLaMA → Mistral), cross-quantization (Q4_K_M → Q8_0), same architecture different sizes (7B → 13B). |
| 4 | **Performance** | Measure save/load latency and throughput. Benchmark against baseline (raw `torch.save`, raw llama.cpp slot save). | Model sizes: 7B, 13B, 70B. Context lengths: 4K, 32K, 128K. Backends: NVMe SSD, HDD, S3. |
| 5 | **Stress** | Rapid save/load cycles, concurrent access, resource exhaustion. | 1000 save/load iterations, multi-process concurrent saves, filesystem full during save, OOM during load. |

### 6.2 Performance Targets

| Scenario | Metric | Target |
| --- | --- | --- |
| 7B model, 4K context, NVMe | Save latency | < 2 seconds |
| 7B model, 8K context, NVMe | Save latency | < 100ms (async path) |
| 70B model, 32K context, NVMe | Save latency | < 10 seconds |
| Any model, any context | Overhead vs. raw save | < 5% added latency from container/integrity |
| Async save (vLLM path) | Inference throughput impact | < 1% degradation |

### 6.3 CI Pipeline

- **Platform:** GitHub Actions
- **Model fixtures:** Small test models in GGUF format (committed or downloaded in CI)
- **Test suite:** `cargo test` (Rust unit + integration) + `pytest python/tests/` (Python integration)
- **Regression gate:** Determinism tests must pass on every PR
- **Performance tracking:** Benchmark results logged per commit, alerts on regression > 10%
- **Format compliance:** Automated checker validates all generated `.sls` files against the format spec

---

## 7. Development Environment Setup

### 7.1 Prerequisites

| Dependency | Version | Required |
| --- | --- | --- |
| Rust toolchain | 1.75+ | Yes |
| Python | 3.10+ | Yes |
| CUDA toolkit | 12.x | No (required for GPU-accelerated save/load and vLLM integration) |
| llama.cpp | Latest | No (required for llama.cpp integration; submodule or system install) |
| maturin | 1.x | Yes (for building PyO3 bindings) |

### 7.2 Repository Structure

```
sls/
├── crates/
│   ├── sls-core/          # Format, reader, writer, integrity engine
│   ├── sls-cli/           # Command-line tool
│   └── sls-py/            # PyO3 bindings
├── python/
│   └── sls_transformers/  # HuggingFace integration package
├── plugins/
│   ├── llama-cpp/         # llama.cpp server extension
│   └── vllm/              # vLLM plugin
├── tests/
│   ├── determinism/
│   ├── integrity/
│   ├── compatibility/
│   ├── performance/
│   └── fixtures/          # Small test models, sample .sls files
├── spec/
│   └── FORMAT.md          # Binary format specification
├── docs/                  # User and contributor documentation
├── Cargo.toml             # Workspace manifest
├── pyproject.toml
└── README.md
```

### 7.3 Build Commands

| Task | Command |
| --- | --- |
| Build Rust crates (release) | `cargo build --release` |
| Build Python bindings (dev) | `maturin develop` |
| Run Rust tests | `cargo test` |
| Run Python tests | `pytest python/tests/` |
| Build CLI binary | `cargo build --release -p sls-cli` |
| Install Python package (local) | `pip install -e python/` |
| Run format compliance checks | `cargo test -p sls-core --test format_compliance` |

---

## 8. Risk Register

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| Model architecture changes break saved states | High | High | Architecture hash includes version. Fail-fast on mismatch with descriptive error. Document migration path for format version bumps. Consider embedding architecture version in hash inputs. |
| KV cache format differs across quantization methods | High | Medium | Abstract over quantization in serialization layer. Store raw bytes with quantization metadata in header. Quantization identifier enables format-aware deserialization. |
| Multi-GB state files create storage and bandwidth pressure | Medium | High | zstd compression (1.3–1.8× on tensor data). Delta saves for incremental changes. Partial serialization for selective layers. Cloud storage backends with streaming upload/download. |
| Malicious `.sls` files exploit deserialization | Medium | Critical | Strict format validation before any data is used. No arbitrary code execution — format contains only raw data, not executable payloads. Sandboxed mmap. Extensive fuzz testing with AFL/libFuzzer. Size limits on sections. |
| vLLM / HuggingFace / llama.cpp diverge on cache internals | High | Medium | Abstraction layer in SLS Runtime isolates backend-specific details. Pin to stable public APIs. Maintain per-backend adapters with independent version tracking. |
| Rust language barrier for ML community contributors | Medium | Medium | Python-first API surface. Rust internals are an implementation detail. Good documentation, contributor guide, and "good first issue" labels on self-contained Rust tasks. |
| Performance regression on async I/O path | Low | Medium | Benchmark suite in CI with regression alerts. Fallback to synchronous path. Runtime detection of `io_uring` availability (kernel version check). |

---

## 9. Success Metrics

| Phase | Metric | Target |
| --- | --- | --- |
| **Phase 0** | Deterministic round-trip | Bit-identical output on LLaMA 3 8B at 4K context |
| **Phase 0** | Save latency | < 2 seconds on consumer NVMe SSD |
| **Phase 1** | Format compliance | All generated files pass spec compliance tests |
| **Phase 1** | CLI file handling | Handles files up to 50 GB without failure |
| **Phase 1** | Crate publication | `sls-core` published on crates.io |
| **Phase 2** | HuggingFace coverage | Works with all Cache subclasses (`DynamicCache`, `StaticCache`, `QuantizedCache`, `OffloadedCache`) |
| **Phase 2** | Installation | `pip install sls-transformers` works on Linux and macOS |
| **Phase 2** | Overhead vs. raw `torch.save` | < 5% added latency |
| **Phase 3** | Server save latency (7B / 8K) | < 100ms (async path) |
| **Phase 3** | REST API completeness | All `/v1/states/*` endpoints functional in both llama.cpp and vLLM |
| **Phase 4** | Fork tree navigation | 10+ branches navigable via `sls log` |
| **Phase 4** | Encryption round-trip | Encrypted states restore correctly with correct key, fail with wrong key |
| **Phase 4** | Cloud backend | S3-compatible backend operational with multipart upload |

---

## 10. Open Implementation Questions

The following questions are unresolved and require discussion before or during implementation. They are listed here to prevent premature decisions from being baked into the format or runtime.

- **Streaming vs. atomic writes:** Should the format support streaming writes (append sections incrementally as they are serialized) or require atomic writes (assemble complete file in memory or temp file, then rename)? Streaming enables lower peak memory usage; atomic writes are simpler to reason about for integrity.

- **LoRA adapter state handling:** Should adapter weights be serialized inline in the `ADAPTER_WEIGHTS` section, or should the section contain only an adapter identifier (name + hash) that references an external adapter file? Inline is self-contained but bloats the file; external reference is smaller but creates a dependency.

- **Token history scope:** Should the `SAMPLER_STATE` section include the full token history (all token IDs generated so far) or just the RNG state + sampling parameters? Full history enables reconstruction of the conversation but increases section size linearly with context length. The `TOKEN_BUFFER` section already stores token IDs — is duplication acceptable for self-contained sampler replay?

- **Delta save granularity:** What is the right granularity for delta saves — per-block (most granular, highest bookkeeping overhead), per-layer (moderate), or per-section (coarsest, simplest)? The answer likely depends on the inference engine: vLLM's block allocator has natural per-block dirty tracking, while HuggingFace's cache is per-layer.

- **Unix pipe support:** Should the CLI support piping via stdin/stdout (e.g., `sls save --output - | gzip > state.sls.gz`)? This enables integration with Unix toolchains but conflicts with mmap-based reading and requires the format to be streamable without seeking.

- **Distributed inference state:** How should the format handle tensor-parallel KV cache split across multiple GPUs? Options include: one `.sls` file per GPU rank (simple but requires orchestration), or a single file with rank-indexed sub-sections in the `KV_CACHE` payload (complex but self-contained).

> **Note**
>
> These questions should be tracked as GitHub issues and resolved through RFC-style discussion with implementation prototypes where the answer is not obvious from first principles.

---

*SLS Implementation Guide v0.1 — April 2026*
*This document is a living specification. File issues and proposals at the project repository.*