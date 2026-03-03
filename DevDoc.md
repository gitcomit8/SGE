
# **SGE — Milestone Roadmap & Formal Invariants (Developer Edition)**

---

> Purpose: a practical, developer-facing milestone document that converts architecture ambition into atomic, testable steps. Each milestone lists the exact invariants that must be *proven* before progressing. No soft opinions — only measurable acceptance criteria, required artifacts, and remediation paths.

---

# Table of Contents

* ### Introduction & How to Use This Document
* ### Progression Rules (gating)
* ### Repository & CI Foundation
* ### Milestone 0 — Deterministic Heartbeat (split)
* ### Milestone 1 — Minimal GPU Pipeline (split)
* ### Milestone 2 — ECS: Sparse-Set Core (split)
* ### Milestone 3 — Voxel Data Model & Naïve Mesher
* ### Milestone 4 — Greedy Meshing, Uploads & Chunk Isolation
* ### Milestone 5 — Visibility, Camera & Culling
* ### Milestone 6 — Job System & Safe Parallel Meshing
* ### Milestone 7 — Render Graph (incremental) & Bindless Experimentation
* ### Milestone 8 — Stress Testing & Telemetry
* ### Milestone 9 — Developer Ergonomics & VFS / Asset System
* ### Milestone 10 — Nice-to-have / Optional (glTF, PBR, hot-reload)
* ### Appendix: Tools, Benchmarks, Tests & Example Commands
* ### Failure Modes & Remediation

---

# Introduction & How to Use This Document

This is a checklist and contract with yourself as the SGE developer.
For each milestone you will produce:

* runnable artifacts (code + a script that runs the acceptance tests),
* a test log (plain-text/perhaps JSON) proving invariants, and
* at least one benchmark showing the metrics claimed.

If any invariant fails, stop. Diagnose. Fix or roll back design decisions. Do not proceed until **all** invariants for that milestone are proven.

---

# Progression Rules (gating)

1. **Gated progression**: You may not start the next milestone until all invariants of the current milestone are satisfied and committed (tests + artifacts in repo).
2. **Measure before change**: For any non-trivial change, capture baseline metrics (frame time, mem usage, allocation counts).
3. **Fail fast and loudly**: Tests should return non-zero exit codes and produce logs for failures.
4. **Documentation required**: Each milestone gets a short `MILESTONE.md` describing decisions and why a chosen invariant exists.
5. **CI**: Basic CI must run unit tests + selected integration tests on Linux (and at least one other platform before milestone 7).
6. **Determinism is king**: Determinism invariants must be validated via byte/bit-level or hash comparisons.

---

# Repository & CI Foundation (Prerequisite)

**Objective**: Provide reproducible build + test environment so all invariants can be validated anywhere.

**Scope**

* Cargo workspace skeleton with crates: `sge-core`, `sge-renderer`, `sge-ecs`, `sge-math`, `sge-voxel`, `sge-tools` (tests/bench harnesses).
* README with bootstrap, build, and run steps.
* CI pipeline: `cargo test`, `cargo build --release`, `cargo bench` (placeholder) on Linux + macOS/Windows matrix.

**Deliverables**

* `./scripts/bootstrap.sh` (installs rustup, sets toolchain)
* `./ci/*` workflow files (or equivalent)
* `MILESTONE-REPO.md` describing CI gating

**Invariants to Prove**

1. `cargo test` passes on CI (exit 0).
2. `cargo build --release` completes without warnings considered errors (lint failures optional per policy).
3. Reproducible build: `git clean -fdx && ./scripts/bootstrap.sh && cargo build --release` produces same binary checksum across two fresh envs (hash in repo).

**Acceptance artifacts**

* CI logs URL or saved CI output file.
* `build-checksum.txt` with two reproducible run hashes.

---

# Milestone 0 — Deterministic Heartbeat (split into atomic sub-achievements)

## 0.A — Window + Loop Skeleton

**Objective**: Window creation and a main loop with clear phases.

**Scope**

* Integrate `winit` for a window.
* Main loop with explicit phases: Input → Update → Render → Present.

**Invariants**

1. Phase separation enforced by code structure (no calling render functions during input pass). *Prove*: provide code map and a small unit test that asserts correct call order via injected mock hooks.
2. The loop has a fixed timestep configuration option (e.g., `dt = 1/60`).

**Acceptance**

* `scripts/run_heartbeat` that runs for 1000 frames and writes a deterministic trace of phase timestamps.

## 0.B — Deterministic Simulation Step

**Objective**: Determinism for pure simulation given identical inputs.

**Scope**

* RNG seeded at start.
* No uncontrolled global state in simulation step.

**Invariants**

1. **Simulation Hash Invariance**: Running the simulation for N ticks with identical inputs produces identical state hash (e.g., XXHash64) across multiple runs and machines.
   *Prove*: provide `scripts/determinism_check` that runs twice and compares the hash.
2. **No non-deterministic OS dependencies** (timestamps, thread scheduling): simulation step must not read clock during deterministic phase.

**Acceptance**

* `determinism.log` showing identical hash values across at least 3 runs.

## 0.C — Frame Pacing & Allocation Invariants

**Objective**: Ensure frame pacing and no per-frame heap allocations in the core loop.

**Invariants**

1. **Frame Jitter**: On baseline hardware, frame time jitter within ±1 ms over 1000 frames. Produce `frame_timing.csv`.
2. **No per-frame heap allocation**: Use a global allocator hook or instrumentation to show zero `malloc`/`alloc` calls during steady-state loop. Provide `alloc_profile.txt`.

**Acceptance**

* Bench logs and allocator instrumentation output.

---

# Milestone 1 — Minimal GPU Pipeline (granular)

## 1.A — `wgpu` Initialization & Backend Validation

**Objective**: Initialize `wgpu` with validation enabled.

**Invariants**

1. Launch with `WGPU_BACKEND=Vulkan` (and fallback) with validation enabled; no validation messages when rendering a static triangle for 10k frames.
2. Provide `wgpu_init.log` that captures debug messages (must be empty of warnings/errors).

## 1.B — Swapchain & Single Render Pass

**Objective**: Render a test triangle/cube with stable presentation.

**Invariants**

1. Rendered output visible, no frame drops over 10k frames.
2. No resource leaks reported at shutdown by GPU validation.

## 1.C — Resource Ownership & Lifetime

**Objective**: Explicit resource lifetimes and deterministic destruction order.

**Invariants**

1. All GPU resource creation and destruction paths are explicit and exercised by tests.
2. Provide a small fuzz on create/destroy order that still produces no validation errors.

**Acceptance**

* `gpu_validation_report.txt`
* screenshot PNG of triangle and log of 10k frame run.

---

# Milestone 2 — ECS: Sparse-Set Core (more atomic)

## 2.A — Entity ID Allocator

**Invariants**

1. Unique stable IDs for at least 10M alloc/free cycles without ID reuse within a configurable generation window.
2. Tests: allocate N, free a subset, allocate again; ensure ID generation semantics hold.

## 2.B — Component Sparse-Set Storage

**Invariants**

1. Component lookup must average O(1) (microbench shown).
2. Contiguous storage for components (no sliced vectors): provide memory layout dump and proof.

## 2.C — Iteration & Query

**Invariants**

1. Iterating 1M components completes under measured threshold (provide numbers).
2. Query consistency: iteration order deterministic.

**Acceptance**

* `ecs_benchmarks.json` with lookup/iteration timings.
* Unit tests asserting invariants.

---

# Milestone 3 — Voxel Data Model & Naïve Mesher

## 3.A — Chunk Model & Storage

**Objective**: Implement chunk container, persistent serialization format, and memory accounting.

**Invariants**

1. Chunk memory footprint calculable and bounded. Provide formula and measured values for a 16×16×16 chunk.
2. Serialization round-trip identity: loading a saved chunk produces same chunk bits.

## 3.B — Naïve Meshing (face-by-face)

**Objective**: Produce a correct mesh from chunk data and render it.

**Invariants**

1. Mesh correctness verified against ground truth (pixel-perfect small scenes or geometry hash).
2. Upload to GPU and render without validation errors.

**Acceptance**

* `chunk_mem_report.txt`
* small demo world and screenshot.

---

# Milestone 4 — Greedy Meshing, Upload Pipeline & Chunk Isolation

## 4.A — Greedy Mesher Implementation

**Invariants**

1. Face-count reduction: greedy meshing reduces face count by >=50% vs naive for test patterns (document test case).
2. Mesh vertex/index buffers canonicalized (no duplicate vertices where possible).

## 4.B — CPU → GPU Upload Pipeline

**Invariants**

1. Batching: uploads for N chunk meshes should be grouped to maintain throughput; measure upload latency.
2. No per-mesh pipeline stalls: present no frame hitch during streaming upload (bench against baseline).

## 4.C — Chunk Isolation & Remesh Scope

**Invariants**

1. Mutating block in chunk A must not remesh chunk B unless adjacent border rules require it.
2. Provide unit test: mutate 100 chunks randomly and prove remesh count ≤ expected local neighborhood count.

**Acceptance**

* `mesher_stats.csv` and remesh logs.

---

# Milestone 5 — Visibility, Camera & Culling (incremental)

## 5.A — Camera System & Frustum Culling

**Invariants**

1. No visible chunk incorrectly culled. Test with labeled chunks and camera positions: assert none of visible labels are missing.
2. CPU cost: frustum culling overhead per frame measured and bounded.

## 5.B — LOD & Distance Culling (if chosen)

**Invariants**

1. LOD transitions must be stable (no popping under normal movement).
2. Visible set size proportional to frustum size, not total world.

**Acceptance**

* `culling_tests.log` and visual debug overlay dumps.

---

# Milestone 6 — Job System & Safe Parallel Meshing (atomic steps)

## 6.A — Job Scheduler (single-producer, multi-consumer)

**Invariants**

1. Bounded queue depth under stress test.
2. Jobs have deterministic completion semantics (when required).

## 6.B — Thread Safety: Meshing Workers

**Invariants**

1. No data races: verify using ThreadSanitizer (TSAN) or equivalent. Provide TSAN logs showing no races across runs.
2. Correctness: parallel meshing results identical to single-threaded meshing (byte-level equality or mesh-equivalence).

## 6.C — Deterministic Integration with Main Thread

**Invariants**

1. Simulation determinism preserved (mesh generation asynchronous only; commit points deterministic).
2. Job cancellation and resubmission semantics well-defined (tests for cancellation).

**Acceptance**

* TSAN logs, bench showing near-linear speedup up to N cores, job queue metrics.

---

# Milestone 7 — Render Graph (incremental) & Bindless Experimentation

## 7.A — Minimal Render Graph

**Objective**: Pass-based dependency graph with deterministic pass ordering.

**Invariants**

1. All resource dependencies must be explicit and graph-validated at build-time.
2. No implicit side-effects between passes (prove with unit tests that reorder-pass fails graph validation).

## 7.B — Descriptor Lifetime & Reuse

**Invariants**

1. Descriptor allocation metrics scale sublinearly with entity count (prove with a test that increases entity resources).
2. No per-frame descriptor explosion.

## 7.C — Bindless Approaches (wgpu-constrained)

**Invariants**

1. Implement a workable approach (e.g., texture arrays, bindless-like descriptors where available) with fallbacks.
2. Test across at least two backends (Vulkan and DX12 or Metal) for compatibility; provide compatibility matrix and notes.

**Acceptance**

* Render graph tests pass on CI platform(s) and logs showing pipeline switch counts.

---

# Milestone 8 — Stress Testing & Telemetry

## 8.A — Large-World Stress Test

**Invariants**

1. Sustained run of 30 minutes without memory growth beyond steady state.
2. No frame spikes > 2× baseline average for normal streaming operations.

## 8.B — Telemetry Dashboard

**Invariants**

1. Continuous metrics for frame/render/sim/meshing/job queue/alloc count are logged and can be plotted.
2. Alerting rules for regression (e.g., median frame time increased by >10%).

**Acceptance**

* `stress_report.zip` containing logs, sample dashboard images, and a short analysis.

---

# Milestone 9 — Developer Ergonomics & VFS / Asset System

## 9.A — Virtual File System

**Invariants**

1. VFS abstraction with mount points and deterministic resolution.
2. Asset reload test: reload a changed asset at runtime without leaking references.

## 9.B — Debugging / Instrumentation UI (minimal)

**Invariants**

1. Debug UI overlay showing realtime metrics and ability to toggle culling/meshing debug views.
2. Logging system supports structured logs and log-level toggles.

**Acceptance**

* Demo video or recorded GIF of asset reload and debug UI usage.

---

# Milestone 10 — Optional / Nice-To-Have (glTF, PBR, Hot-Reload)

These are explicitly *optional* and gated behind milestone 8 success.

**Invariants**

* glTF loader must decode common glTF 2.0 models and produce GPU resources without validation warnings.
* PBR pipeline must be benchmarked to confirm performance expectations.
* Hot-reload: script or shader reload works without memory leaks or state corruption.

---

# Appendix: Tools, Benchmarks, Tests & Example Commands

**Suggested tests / tools**

* Determinism: deterministic RNG + hash (xxhash / blake2b) of serialized state.
* Alloc instrumentation: custom global allocator in Rust or `jemalloc` stats.
* Concurrency: ThreadSanitizer (TSAN) and Miri for UB checks (Miri on smaller units).
* GPU validation: enable `wgpu`/Vulkan validation layers and capture logs.
* CI: GitHub Actions matrix: `ubuntu-latest`, `windows-latest`, `macos-latest`.
* Bench harness: `criterion` or `cargo bench` for microbenchmarks.

**Example commands**

* Run deterministic test: `./scripts/determinism_check`
* Run heartbeat: `./scripts/run_heartbeat --frames 1000`
* Run stress test: `./scripts/stress_world --chunks 5000 --duration 1800`
* Run TSAN (nightly): `RUSTFLAGS="-Z sanitizer=thread" cargo +nightly test --tests`

(Adjust commands to your repo layout; scripts should be committed as part of each milestone.)

---

# Failure Modes & Remediation

If an invariant fails:

1. **Record** full logs and stop. Tag the failing commit.
2. **Roll back** if the failure is due to a risky refactor; otherwise apply targeted fix.
3. **Hypothesis-driven debugging**: Clearly state the hypothesis, the test that will confirm it, and the remediation plan in `MILESTONE.md`.
4. **If determinism breaks**: bisect to single commit; check RNG seeding, global state, and serialization order.
5. **If performance regresses**: revert to the last known good baseline; add more granular microbenchmarks and performance regression tests.

---

# Final Notes (developer guidelines)

* Make each milestone produce *evidence*. Evidence = test scripts, logs, benchmark numbers, and a short written note explaining the outcome.
* Keep changes small and measurable. Big rewrites are allowed only when a milestone’s core invariants are impossible to reach otherwise, and only after a documented design review.
* Treat SGE as a performance lab, not an SDK-first product. Design for measurement and revision.

---

If you want, I’ll generate the `MILESTONE.md` file template (copy/paste ready) that you can drop into each milestone folder, with fields for artifacts, commands, metrics, and a checklist for required logs.
