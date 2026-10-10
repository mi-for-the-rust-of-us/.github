# Roadmap

The cross-crate view for [mi-for-the-rust-of-us](https://github.com/mi-for-the-rust-of-us): what is shipped, what is next, and the work that belongs to no single crate.

**This file is not the authority on any individual crate.** Each repository keeps its own `CHANGELOG.md`, which is the record of what actually shipped, and each keeps a detailed roadmap of its own (a root `ROADMAP.md`, or `docs/roadmaps/` for hf-fetch-model). This page exists for the questions those files structurally cannot answer: how the five crates constrain each other, and which pieces of work cut across all of them.

## Where things stand

| Crate | Released | MSRV | Next |
|---|---|---|---|
| [candle-mi](https://github.com/mi-for-the-rust-of-us/candle-mi) | `0.2.1` | 1.91 | `0.2.2`, in preparation: the `fused-attn` feature (`OthelloGpt`'s attention through candle-fused-attn) and a supply-chain hardening of the release pipeline. Then the parked `0.3.0` API-ergonomics batch |
| [hf-fetch-model](https://github.com/mi-for-the-rust-of-us/hf-fetch-model) | `0.12.1` | 1.91 | A patch already on `main`: the anamnesis `0.7.10` bump (`inspect`'s dequantised size for 4-bit GPTQ / AWQ was 8x too small), the cache commands' disk-use under-count ([#16](https://github.com/mi-for-the-rust-of-us/hf-fetch-model/issues/16): they counted `snapshots/`, never `blobs/`), and release-pipeline hardening. Then `0.13.0`: a multi-GPU `--check-gpu` matrix, gated on second-GPU hardware |
| [anamnesis](https://github.com/mi-for-the-rust-of-us/anamnesis) | `0.7.10` | 1.88 | `0.8.0`: Python bindings (Phase 8). Then `0.8.5`: Lethe encode completion |
| [hypomnesis](https://github.com/mi-for-the-rust-of-us/hypomnesis) | `0.2.14` | 1.88 | `0.3.0`, mostly gated candidates, with one change decided (JSON `spilled` becomes `null` when unmeasurable) and one un-gated (a unified spill-condition core) |
| [candle-fused-attn](https://github.com/mi-for-the-rust-of-us/candle-fused-attn) | `0.4.0` | 1.88 | Candidates, not yet chosen: why the A100 is slower than PyTorch's SDPA (a profile with counters, then a per-card occupancy lever; an opt-in 3xTF32 path for sm_80 / sm_90 if the gap remains), and larger tiles at 8 warps per SM on consumer cards, through swizzled shared memory |

All five are pre-1.0 and all five are on **edition 2024**. APIs may change between minor versions.

## Cross-cutting work

These are the items that no single repository owns, listed roughly by how much they change for people outside this organization.

### Python reach

anamnesis `0.8.0` ships `pip install anamnesis-quant` via PyO3 and maturin (the distribution name differs from the crate's; anamnesis's roadmap says why). The [interop contract](https://github.com/mi-for-the-rust-of-us/anamnesis/blob/main/docs/python-interop.md) is fixed before the bindings exist rather than retrofitted after: typed exceptions, owned NumPy arrays, and `ml_dtypes.bfloat16` when that package is importable, for callers who ask for bf16 at all, since the output dtype is now the caller's choice.

This is the single largest audience change on the roadmap, and it is worth being precise about why it sits on anamnesis rather than candle-mi. Dequantization and format conversion are a problem Python already has and solves slowly. Mechanistic interpretability in Rust is a problem most Python users do not have. The crate that should cross the language boundary first is the one whose value does not depend on the reader adopting Rust.

### Dictionary training

Reference-grade TopK sparse-autoencoder training is planned, as a sibling crate rather than inside candle-mi, and has no version assigned yet. candle-mi is not a training crate: training capability lives beside it, the way candle-fused-attn carries the trainable attention. Today candle-mi can *load and use* SAEs, CLTs and PLTs, and nothing on this stack can *train* them, which means every experiment is limited to dictionaries somebody else published, for models somebody else chose.

Masked-diffusion models are the proving ground, because no public dictionary exists for them at all, so an SAE trained on this stack is the only way DLM-Scope-style analysis happens.

The groundwork is already carrying real weight rather than waiting to be tested. The trainable-backbone work of v0.1.20 through v0.1.22 (`track_op` dispatch, always compiled, and `OthelloGpt::init`, under `diffusion`, in v0.1.20; the checkpointable `AdamW` in v0.1.21; then `fold_ema` and a single-launch multi-tensor `AdamW` on CUDA in v0.1.22; the optimizer pieces sit behind the `training` feature) is what a multi-epoch, multi-stage masked-diffusion training run is currently running on, staged across process boundaries and validated step-for-step against a PyTorch oracle. The same run is where candle-fused-attn came from: its step measured 5.1× slower than the PyTorch reference on 2026-07-29 (RTX 5060 Ti, batch 64), and 0.90× the PyTorch time on 2026-10-04 with candle-fused-attn 0.3.0 (RTX 5090, batch 128). The SAE trainer lands on tested ground.

### Edition 2024, done

All five crates are on edition 2024. anamnesis migrated in `0.7.3` and hf-fetch-model in `0.11.3`, one crate at a time as planned; candle-fused-attn started there.

Worth keeping the reason it was treated as a real migration rather than a manifest edit. The `rust_2024_compatibility` lint group surfaced exactly one issue class in both crates, and it was the one that does not announce itself: **`tail_expr_drop_order`**, relative drop order changing in Rust 2024. One site in anamnesis's threaded dequant dispatch (a `handle.join()`), nineteen across hf-fetch-model's async stack, every one a `while let … .await` loop over a `JoinSet`, a stream or a response's chunks, or an async function's tail. Each site was read, and none held a lock, a temp-file guard or a semaphore permit across the changed drop point, so nothing changed behaviour. Neither crate was exposed to the usual hazards: `unsafe_op_in_unsafe_fn` and `static_mut_refs` did not apply. The risk was never compilation; it was silent behaviour change in concurrent code, where the compiler can say where to look but not whether it matters, which is why every site was read rather than trusted.

### Release lockstep

This is the constraint most likely to bite someone who edits one manifest without reading another.

`candle-mi` and `hf-fetch-model` both depend on `anamnesis`, and `candle-mi` depends on `hf-fetch-model`. If the two `anamnesis` requirements ever land in different semver-compatible ranges, cargo resolves **two copies** of anamnesis into the same tree, and the format types stop being the same types. Any anamnesis version bump therefore has to move through `hf-fetch-model` and `candle-mi` together, not independently. The same hazard produced the `windows-sys` deduplication in candle-mi v0.1.18.

`hypomnesis` is the loose one: candle-mi takes it optionally behind the `memory` feature, hf-fetch-model requires it, and nothing in the public API crosses between them, so its releases do not need coordinating.

`candle-fused-attn` couples on a different axis: not on any crate of this organization, but on **candle** itself. Its only dependency is `candle-core` 0.11, and candle-mi will take it optionally behind a `fused-attn` feature from `0.2.2`. Its ops are `CustomOp`s on candle-mi's own tensors, so if the two ever resolved different candle minors, cargo would build two `candle-core`s and the tensors would stop being the same type. A candle `0.12` bump therefore moves through candle-fused-attn first, then candle-mi.

A second, subtler coupling surfaced with hypomnesis `0.2.8`, which made `cli` a **default** feature. hf-fetch-model was taking hypomnesis with defaults on, and because cargo unions features across the graph, that dragged `clap` (four crates), `anstyle` and `ctrlc` into **candle-mi's** build regardless of candle-mi's own `default-features = false`. Fixed in hf-fetch-model 0.11.3. The general lesson: **an optional-by-default feature in a leaf crate is not local to that crate**, and a `default-features = false` opt-out only holds if every path to the dependency also opts out.

### Documentation and CI hygiene

- **Per-feature rustdoc lane: done.** Deferred from candle-mi v0.1.21, which found five broken intra-doc links that no existing lane could catch; candle-mi's `ci.yml`, `publish.yml` and `preflight.ps1` now all run rustdoc per feature. One gap is already known for `0.2.2`: as prepared, its `fused-attn` feature gets a `diffusion,fused-attn` lane in `ci.yml` and preflight, but none in `publish.yml`, for rustdoc or for clippy.
- **The docs.rs feature list: still unchecked.** candle-mi's `[package.metadata.docs.rs]` list had silently omitted `training`, so `optim::AdamW`, `fold_ema` and `FoldPath` shipped in 0.1.22 with **no docs.rs pages at all**. v0.1.23 added it back, and the list is about to drift again: as prepared for `0.2.2`, it omits `fused-attn`, harmlessly because that feature gates no public item. Nothing checks that the list covers the feature set, which is the actual defect.
- **Clippy on test and example targets: done, but not gating.** Since v0.2.0 every candle-mi clippy lane passes `--all-targets`, across twelve feature lanes. The lanes still run `-W clippy::pedantic` rather than `-D`, so a pedantic finding in a test or example is a warning, not a failure. Making it gate means respecting two measurement traps, or the lane will lie: cargo does not re-emit diagnostics for **fresh units** (a warm `target/` under-reports, so local preflight disagrees with a cold CI runner), and `-D warnings` **aborts the build before later targets compile**, so a gating lane reveals findings in waves rather than all at once.
- **Every crate now has an MSRV lane** alongside rolling `stable`. hf-fetch-model was the last gap, and closing it immediately surfaced a real failure, a clippy lint that fired on the MSRV toolchain and not on stable. A stable-only CI cannot see version drift in either direction.

  The floors are no longer uniform, and that is dependency-driven rather than a choice: **1.88** for anamnesis, hypomnesis and candle-fused-attn (whose `candle-core` 0.11 pulls `zip` 8.6, itself needing 1.88), **1.91** for hf-fetch-model and candle-mi. `hf-hub` 1.0 brings a mandatory `hf-xet`, and inside it `xet-core-structures` declares no `rust-version` at all while calling `str::floor_char_boundary`, stable only since 1.91. Cargo's MSRV resolution therefore reports 1.89 and is wrong: **the declared-metadata floor is a lower bound, and only a real build proves the number.**

### Upstream candle

Two threads, both feeding [huggingface/candle](https://github.com/huggingface/candle).

**The fused-operation cluster.** Nine related issues and pull requests, mapped in a [posted comment on candle#2168](https://github.com/huggingface/candle/issues/2168#issuecomment-5150289607); drafts and the tracker live in candle-mi's `docs/upstream/`.

**Three open pull requests, all from actually training a model on this stack**, all measured on the same RTX 5060 Ti:

| PR | What it does | Effect |
|---|---|---|
| [#3819](https://github.com/huggingface/candle/pull/3819) | `AdamW::step_t`, `set_step_t`, `moments` | Makes optimizer state reachable, so a run staged across processes resumes instead of re-applying Adam's warm-up bias correction at every boundary |
| [#3822](https://github.com/huggingface/candle/pull/3822) | Store the first gradient directly instead of adding it into fresh zeros | `backward()` 413.7 ms to 251.8 ms; whole step 0.521 s to 0.391 s |
| [#3823](https://github.com/huggingface/candle/pull/3823) | Analytic `CustomOp::bwd` for the fused `softmax_last_dim` and `layer_norm` | Step 0.391 s to 0.333 s, 41.9k to 49.2k tokens/s, with step-0 loss bit-identical to PyTorch |

All three are still open with no maintainer review. A triage comment on #3823 (2026-10-01) offers the maintainers the shorter route where one exists: closing #3822 in favour of the earlier, equivalent #3578, and either #3823 or the overlapping #3724 and #3613.

#3823 is a correctness fix before it is a performance one. Those fused ops record no backward op, so `backward()` reaches them and stops: every parameter upstream trains as if frozen, with no error, no warning, and a loss that still goes down.

candle-mi already defends against that without waiting for a merge. Its `nn_ops` module probes once per process whether the installed `candle-nn`'s fused ops actually carry gradients, then dispatches on `Tensor::track_op`: an inference forward takes the fused kernel and stays byte-identical to calling it directly, while a forward under a `VarMap` takes the composed form when the runtime cannot back-propagate through the fused one. Fast when the runtime supports it, correct when it does not.

candle-fused-attn is the same pattern, taken to the size of a crate. candle has no fused attention that can train, and none in fp32 on CUDA: `candle-flash-attn` is f16/bf16 and forward-only, and candle's own fp32 fused paths, on Metal and CPU, are forward-only too. Rather than wait for one, candle-fused-attn ships it as `CustomOp`s over **stock** candle 0.11: no fork, no `[patch]`, nothing to upstream before it is usable.

That is the pattern for everything upstream here: ship the local defense first, then send the fix. Upstream reacts on a scale of weeks to months, so nothing on this roadmap is scheduled behind a merge. Current training runs consume the two performance patches through a `[patch.crates-io]` pointing at a local candle clone; if the PRs land, that patch section is deleted and nothing else changes.

## How this roadmap decides what to build

Two rules, and hypomnesis is the clearest example of both.

**Demand gates features.** hypomnesis's `0.3.0` section is mostly a list of things deliberately *not* built yet: a segmented per-process VRAM API, per-process attribution inside the spill tracker, a `GpuDeviceInfo::fits()` library method. Each one records what would un-gate it, and until that happens it stays unbuilt. Unshipped work with a written reason is more useful than a promise with a date. The gate works in both directions: the unified spill-condition core sat on that list until v0.2.13, when a real run hit the case it covers, and it was un-gated on the spot.

**Dogfooding supplies the demand.** Nine of the ten hypomnesis releases from v0.2.4 to v0.2.13 originate in a written report from a real research run, not from a feature wishlist; the tenth came from a self-audit. `hmn watch` exists because a 15-hour training campaign had no way to attach to a job already running. `--follow-new` exists because `hmn watch` attached to 19 sequential test processes and froze its PID set at the wrong moment. The `cli` feature became default-on because a rented-GPU deploy ran `cargo install hypomnesis`, got exit 0, and got no binary.

The consequence worth stating plainly: **if you want something here, an issue describing the run that needed it will move it much further than a feature request.**

## Not planned

- **An inference engine.** candle-mi recomputes the full sequence at every generation step and ships no KV cache, on purpose, because interventions have to be re-observed at every position. For serving, use [candle-vllm](https://github.com/EricLBuehler/candle-vllm), [vllm.rs](https://github.com/guoqingbao/vllm.rs) or [vLLM](https://github.com/vllm-project/vllm).
- **A training loop inside candle-mi.** This one is a scope boundary, not an absence: training on this stack is active and load-bearing, and it is where all three candle pull requests above came from. candle-mi ships the pieces a loop cannot reconstruct for itself, namely backbones that carry gradients end to end over a `VarMap`, seeded from-scratch initialization so a reference model is reproducible from `(config, seed)` alone, backward-safe fused-op dispatch, a checkpointable `AdamW` whose moments and step counter survive a process boundary, and `fold_ema`, the fused update primitive for a parameter EMA. The loop itself, along with the learning-rate schedule, the EMA's shadow parameters and when to fold them, the decay/no-decay split, the data loader and the prefetch pipeline, lives in the consumer. Today that consumer trains candle-mi's own `OthelloGpt` over a `VarMap`, so the trained object and the probed object are literally the same object and cannot drift apart. Those pieces graduate into candle-mi if a second consumer needs them: one consumer is an experiment, two is an API.
- **GPU backends nobody here can test.** AMD ROCm, Intel Arc and Intel-Mac Metal stay in hypomnesis's carried-forward table, each waiting on hardware access or a contributor PR. Shipping untested FFI is against the discipline these crates are written to.

## Where the detail lives

| Question | Where |
|---|---|
| What shipped, and when | Each crate's `CHANGELOG.md` |
| What a specific crate plans next | Each crate's `ROADMAP.md` |
| Why an interpretability finding came out the way it did | candle-mi's [`docs/experiments/`](https://github.com/mi-for-the-rust-of-us/candle-mi/tree/main/docs/experiments) |
| API reference | docs.rs, linked from every repository sidebar |
