# Roadmap

The cross-crate view for [mi-for-the-rust-of-us](https://github.com/mi-for-the-rust-of-us): what is shipped, what is next, and the work that belongs to no single crate.

**This file is not the authority on any individual crate.** Each repository keeps its own `CHANGELOG.md`, which is the record of what actually shipped, and most keep a detailed `ROADMAP.md` of their own. This page exists for the questions those files structurally cannot answer: how the four crates constrain each other, and which pieces of work cut across all of them.

## Where things stand

| Crate | Released | Next |
|---|---|---|
| [candle-mi](https://github.com/mi-for-the-rust-of-us/candle-mi) | `0.1.21` | `0.2.0`: dictionary training and pre-1.0 API hygiene |
| [hf-fetch-model](https://github.com/mi-for-the-rust-of-us/hf-fetch-model) | `0.11.2` | No dated milestone. Remote header inspection is complete for all four formats; further work is demand-gated |
| [anamnesis](https://github.com/mi-for-the-rust-of-us/anamnesis) | `0.7.2` | `0.7.3`: caller-chosen output dtype. Then `0.8.0`: Python bindings |
| [hypomnesis](https://github.com/mi-for-the-rust-of-us/hypomnesis) | `0.2.8` | Nothing scheduled. `0.3.0` is a list of candidates, every one gated on a real consumer asking |

All four are pre-1.0. APIs may change between minor versions.

## Cross-cutting work

These are the items that no single repository owns, listed roughly by how much they change for people outside this organization.

### Python reach

anamnesis `0.8.0` ships `pip install anamnesis` via PyO3 and maturin. The [interop contract](https://github.com/mi-for-the-rust-of-us/anamnesis/blob/main/docs/python-interop.md) is already frozen: typed exceptions, owned NumPy arrays, `ml_dtypes.bfloat16`.

This is the single largest audience change on the roadmap, and it is worth being precise about why it sits on anamnesis rather than candle-mi. Dequantization and format conversion are a problem Python already has and solves slowly. Mechanistic interpretability in Rust is a problem most Python users do not have. The crate that should cross the language boundary first is the one whose value does not depend on the reader adopting Rust.

### Dictionary training

candle-mi `0.2.0` adds reference-grade TopK sparse-autoencoder training. Today candle-mi can *load and use* SAEs, CLTs and PLTs but cannot *train* them, which means every experiment is limited to dictionaries somebody else published, for models somebody else chose.

Masked-diffusion models are the proving ground, because no public dictionary exists for them at all, so a trained-in-candle-mi SAE is the only way DLM-Scope-style analysis happens. The v0.1.20 and v0.1.21 trainable-backbone work (`track_op` dispatch, `OthelloGpt::init`, the checkpointable `AdamW` behind the `training` feature) exists to make this possible.

### Edition 2024

candle-mi and hypomnesis are on edition 2024. anamnesis and hf-fetch-model are still on 2021, and this is not an oversight, it is a deferred cost that has now been measured.

Running the `rust_2024_compatibility` lint group against both crates surfaces exactly one issue class, and it is the one that does not announce itself: `tail_expr_drop_order`, relative drop order changing in Rust 2024.

- **anamnesis**: 1 site, in the threaded dequant dispatch.
- **hf-fetch-model**: roughly 16 sites, spread across the async stack. The types involved are tokio `Semaphore` permits, tokio `Mutex` guards, `futures-channel` oneshots and `Bytes`, which is to say precisely the destructors whose timing is observable.

Neither crate is exposed to the usual 2024 hazards: `unsafe_op_in_unsafe_fn` and `static_mut_refs` do not apply, and MSRV 1.88 clears the 1.85 floor. The risk is not compilation, it is silent behavior change in concurrent code, and the compiler can only say where to look, not whether it matters. These migrate one crate at a time, deliberately, when there is a reason beyond uniformity. anamnesis is the cheap one.

### Release lockstep

This is the constraint most likely to bite someone who edits one manifest without reading another.

`candle-mi` and `hf-fetch-model` both depend on `anamnesis`, and `candle-mi` depends on `hf-fetch-model`. If the two `anamnesis` requirements ever land in different semver-compatible ranges, cargo resolves **two copies** of anamnesis into the same tree, and the format types stop being the same types. Any anamnesis version bump therefore has to move through `hf-fetch-model` and `candle-mi` together, not independently. The same hazard produced the `windows-sys` deduplication in candle-mi v0.1.18.

`hypomnesis` is the loose one: candle-mi takes it optionally behind the `memory` feature, hf-fetch-model requires it, and nothing in the public API crosses between them, so its releases do not need coordinating.

### Documentation and CI hygiene

- **Per-feature rustdoc lane**, deferred from candle-mi v0.1.21, which found five broken intra-doc links that no existing lane could catch. Wants adding to `ci.yml`, `publish.yml` and `preflight.ps1`.
- **MSRV lanes are now uniform.** All four crates test `1.88` alongside rolling `stable`. hf-fetch-model was the last gap, and closing it immediately surfaced a real failure, a clippy lint that fires on 1.88 and not on stable. A stable-only CI cannot see version drift in either direction.

### Upstream candle

candle-mi tracks a cluster of nine fused-operation issues upstream in [huggingface/candle](https://github.com/huggingface/candle), mapped in a [posted comment on candle#2168](https://github.com/huggingface/candle/issues/2168#issuecomment-5150289607); drafts and the tracker live in candle-mi's `docs/upstream/`. Nothing in this roadmap is scheduled behind an upstream fix. Upstream reacts on a scale of weeks to months, so every issue is written to stand alone and every workaround ships locally first.

## How this roadmap decides what to build

Two rules, and hypomnesis is the clearest example of both.

**Demand gates features.** hypomnesis's `0.3.0` section is a list of things deliberately *not* built: an AMD ROCm backend, a segmented per-process VRAM API, per-process attribution inside the spill tracker. Each one records what would un-gate it, and until that happens it stays unbuilt. Unshipped work with a written reason is more useful than a promise with a date.

**Dogfooding supplies the demand.** Most hypomnesis releases since v0.2.4 originate in a written report from a real candle-mi or askesis run, not from a feature wishlist. `hmn watch` exists because a 15-hour training campaign had no way to attach to a job already running. `--follow-new` exists because `hmn watch` attached to 19 sequential test processes and froze its PID set at the wrong moment. The `cli` feature became default-on because a rented-GPU deploy ran `cargo install hypomnesis`, got exit 0, and got no binary.

The consequence worth stating plainly: **if you want something here, an issue describing the run that needed it will move it much further than a feature request.**

## Not planned

- **An inference engine.** candle-mi recomputes the full sequence at every generation step and ships no KV cache, on purpose, because interventions have to be re-observed at every position. For serving, use [candle-vllm](https://github.com/EricLBuehler/candle-vllm), [vllm.rs](https://github.com/guoqingbao/vllm.rs) or [vLLM](https://github.com/vllm-project/vllm).
- **A training framework.** candle-mi's backbones carry gradients and it ships a checkpointable optimizer, but no training loop, schedule or data loader. Those are experiment-shaped and stay with the caller.
- **GPU backends nobody here can test.** AMD ROCm, Intel Arc and Intel-Mac Metal are all un-gated on hardware access or a contributor PR. Shipping untested FFI is against the discipline these crates are written to.

## Where the detail lives

| Question | Where |
|---|---|
| What shipped, and when | Each crate's `CHANGELOG.md` |
| What a specific crate plans next | Each crate's `ROADMAP.md` |
| Why an interpretability finding came out the way it did | candle-mi's [`docs/experiments/`](https://github.com/mi-for-the-rust-of-us/candle-mi/tree/main/docs/experiments) |
| API reference | docs.rs, linked from every repository sidebar |
