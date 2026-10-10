# MI for the Rust of us

**Mechanistic interpretability for language models, in Rust, on the GPU you already own.**

[![Runs on a single RTX 5060 Ti, 16 GB](https://img.shields.io/badge/runs_on-RTX_5060_Ti_16_GB-76B900?logo=nvidia&logoColor=white)](#the-16-gb-rule)
[![Cluster required: no](https://img.shields.io/badge/cluster_required-no-2a6fdb)](#the-16-gb-rule)

Mechanistic interpretability studies *how* a language model reaches its predictions: not just what it emits, but which internal components caused it. The published work in this field mostly assumes a Python stack and a datacenter GPU. This organization is a bet that it does not have to.

## The 16 GB rule

Every crate here exists because of one constraint: **a single consumer card has to be enough.** The reference machine is an RTX 5060 Ti with 16 GB of VRAM. That is not a minimum spec. It is the entire budget.

The constraint turned out to be far less limiting than the field's tooling assumes. On that one card, at F32 precision, which is what research-grade numerical parity actually requires:

- Decoder-only transformers up to roughly **7B parameters** load and run, including LLaMA 1/2/3, Mistral, Qwen 2/2.5/3, Phi-3/4, Gemma, Gemma 2 and StarCoder2. Also RWKV-6/7 linear RNNs and masked-diffusion language models, which most interpretability tooling does not reach at all.
- Anthropic's [Figure 13 rhyme-planning experiment](https://transformer-circuits.pub/2025/attribution-graphs/biology.html#dives-poem-location) runs end to end on a **524K-feature Cross-Layer Transcoder**, suppress-and-inject position sweep included, against a CLT implementation validated at **90/90 top-10 features** against the Python reference. And not once: enough model-by-transcoder cells fit on the card to show that the single-position signature reproduces while the *planning site itself does not*. That is a finding you cannot reach by replicating a figure one time.
- Every numeric path is checked against a `PyTorch` fp32 oracle, and **the date of the check is in the repository**. candle-mi's [`RESURRECTION.md`](https://github.com/mi-for-the-rust-of-us/candle-mi/blob/main/RESURRECTION.md) carries 23 oracle entries with a per-test last-verified date, re-run locally because CI structurally cannot (gated models, and many need the 16 GB card). For interpretability that matters more than throughput: the product is a claim about a model's internals, so an implementation that quietly differs from the reference does not run slower, it produces a finding that is not there.

None of that needs a cluster, an H100, or a cloud budget. If you have a 16 GB card, most of what is in these repositories is reachable from your desk, and that is the whole point of the organization.

## The eco-system

```mermaid
graph TD
    CM["<b>candle-mi</b><br/>hook, capture, intervene"]
    HFM["<b>hf-fetch-model</b><br/>get the weights"]
    AMN["<b>anamnesis</b><br/>read the weights"]
    HMN["<b>hypomnesis</b><br/>watch the memory"]

    CM --> HFM
    CM -. "sae / stoicheia / quantized" .-> AMN
    CM -. "memory" .-> HMN
    HFM --> AMN
    HFM --> HMN
```

Solid arrows are hard dependencies; dotted arrows are optional, behind the named feature flags.

## The crates

| Crate | | What it is |
|---|---|---|
| **[candle-mi](https://github.com/mi-for-the-rust-of-us/candle-mi)** | [![crates.io](https://img.shields.io/crates/v/candle-mi.svg)](https://crates.io/crates/candle-mi) [![docs.rs](https://docs.rs/candle-mi/badge.svg)](https://docs.rs/candle-mi) | The interpretability toolkit. Transformer, RWKV, masked-diffusion and tiny-model backends re-implemented with type-safe hook points, so you can capture and intervene on activations mid-forward-pass. Since v0.2.0 that second verb holds on **every** backend: the recurrent and tiny-model ones used to ignore interventions silently, which made a causal test return a clean null. Built on [candle](https://github.com/huggingface/candle). 42 runnable examples. |
| **[hf-fetch-model](https://github.com/mi-for-the-rust-of-us/hf-fetch-model)** | [![crates.io](https://img.shields.io/crates/v/hf-fetch-model.svg)](https://crates.io/crates/hf-fetch-model) [![docs.rs](https://docs.rs/hf-fetch-model/badge.svg)](https://docs.rs/hf-fetch-model) | Get models off the Hugging Face Hub and find out what is in them. Multi-connection parallel downloads, cache management, and remote tensor-header inspection over HTTP Range, so you can read shapes, dtypes and a GPU-fit verdict without pulling a single weight byte. CLI: `hf-fm`. |
| **[anamnesis](https://github.com/mi-for-the-rust-of-us/anamnesis)** | [![crates.io](https://img.shields.io/crates/v/anamnesis.svg)](https://crates.io/crates/anamnesis) [![docs.rs](https://docs.rs/anamnesis/badge.svg)](https://docs.rs/anamnesis) | Parse any tensor format, recover any precision. Reads `.safetensors`, `.gguf`, `.npz` and PyTorch `.pth`; dequantizes FP8, GPTQ, AWQ, BitsAndBytes, NVIDIA NVFP4 and all 25 production GGUF block types bit-exactly, to `BF16` by default or `F32` / `F16` on request; converts between formats. Hardened for untrusted input. CLI: `amn`. |
| **[hypomnesis](https://github.com/mi-for-the-rust-of-us/hypomnesis)** | [![crates.io](https://img.shields.io/crates/v/hypomnesis.svg)](https://crates.io/crates/hypomnesis) [![docs.rs](https://docs.rs/hypomnesis/badge.svg)](https://docs.rs/hypomnesis) | External RAM and VRAM measurement for Rust processes. Process RSS plus per-process and device-wide GPU memory (Windows DXGI, NVML and PDH; Linux NVML; macOS libSystem and Metal), and WDDM spill detection that tells you when your run started paging into system RAM. CLI: `hmn`. |
| **[candle-fused-attn](https://github.com/mi-for-the-rust-of-us/candle-fused-attn)** | [![crates.io](https://img.shields.io/crates/v/candle-fused-attn.svg)](https://crates.io/crates/candle-fused-attn) [![docs.rs](https://docs.rs/candle-fused-attn/badge.svg)](https://docs.rs/candle-fused-attn) | Fused fp32 scaled-dot-product attention for candle, forward **and** backward, with a bit-reproducible backward: the FlashAttention-2 algorithm in plain fp32 CUDA, as custom ops over stock candle 0.11. It exists because candle has no fused attention that can train, and none in fp32 on CUDA. On consumer cards it beats PyTorch's fp32 SDPA (RTX 5060 Ti: 3.73 against 4.62 ms forward + backward) at half its error against fp64; on an A100, SDPA's 3xTF32 tensor-core path is 1.7× faster. Falls back to a composed CPU reference, so its gradient tests run anywhere. |

## Start here

- **I want to see inside a model I can run locally.** Attention patterns, logit lens, activation patching, steering, cross-layer transcoders: [candle-mi](https://github.com/mi-for-the-rust-of-us/candle-mi), then its [examples](https://github.com/mi-for-the-rust-of-us/candle-mi/blob/main/examples/README.md).
- **I want to know whether a model will fit before I download 15 GB.** `hf-fm inspect <repo> --check-gpu`: see [hf-fetch-model](https://github.com/mi-for-the-rust-of-us/hf-fetch-model).
- **I have a quantized or awkward checkpoint and I need real tensors out of it.** `amn remember <file>`: see [anamnesis](https://github.com/mi-for-the-rust-of-us/anamnesis).
- **My training or inference run is mysteriously slow and I suspect VRAM spill.** `hmn watch <pid>`: see [hypomnesis](https://github.com/mi-for-the-rust-of-us/hypomnesis).
- **I train in candle and need fp32 attention gradients that rerun bit-identical, without paying for the composed version.** `candle_fused_attn::fused_attention_qkv`: see [candle-fused-attn](https://github.com/mi-for-the-rust-of-us/candle-fused-attn) and its [tutorial](https://github.com/mi-for-the-rust-of-us/candle-fused-attn/blob/main/docs/tutorial.md).

**Four of these five are not about interpretability at all.** anamnesis, hf-fetch-model, hypomnesis and candle-fused-attn carry no MI concepts in their APIs. They exist because building an MI toolkit on consumer hardware, and then training the models it studies, kept running into the same unsolved problems, and each one turned out to be worth solving on its own terms. If you write Rust against `burn`, `tch` or `candle` and have never touched interpretability, the bottom four crates are still for you; candle-fused-attn in particular is for anyone training in `candle`.

## What the five crates share

- **A tested MSRV on every crate**, each with its own CI lane alongside rolling `stable`, so a compiler that moves under us cannot quietly break the floor in either direction. The floor is **1.88** for anamnesis, hypomnesis and candle-fused-attn, and **1.91** for hf-fetch-model and candle-mi, which reach `hf-xet` through `hf-hub` 1.0. Both numbers are what the dependency graph enforces, not a preference.
- **MIT OR Apache-2.0**, on every crate.
- **`unsafe` is forbidden by default.** hf-fetch-model forbids it outright. The other four allow it only in the narrow paths that genuinely need it (memory-mapped reads, FFI to NVML, DXGI, PDH and Metal, and CUDA kernel launches), and each states its own exemption in its README badge.
- **Pre-1.0.** The APIs may change between minor versions. Every crate keeps a `CHANGELOG.md` following [Keep a Changelog](https://keepachangelog.com/), and that changelog, not any roadmap, is the authoritative record of what shipped.
- **Dogfooding is the development method.** candle-mi is the demanding consumer that drives hf-fetch-model, anamnesis and hypomnesis: most of hypomnesis's releases since v0.2.4 originate in a written dogfooding report from a real research run, not from a feature wishlist.

  The clearest single measurement of that loop runs between two of these crates. candle-mi v0.2.0 adopted hf-fetch-model's HTTP Range inspect to read a `GemmaScope` transcoder's dimensions, and the same file on the same link went from **302,131,416 bytes in 61.97 s** to **86,037 bytes in 3.63 s**: eight range requests, 0.028% of the archive, for the two integers it actually needed. The five findings that integration produced went back upstream as a report. That is the whole thesis in one line, and it is measured rather than asserted.

  candle-fused-attn is the same loop one level further down. It was written for a real training run, a masked-diffusion model trained through candle-mi, whose step measured **5.1× slower than its PyTorch reference on 2026-07-29** (RTX 5060 Ti, batch 64). With candle-fused-attn 0.3.0 the same step runs at **0.90×** the PyTorch time (RTX 5090, batch 128).

## Roadmap

The cross-crate roadmap lives in [ROADMAP.md](https://github.com/mi-for-the-rust-of-us/.github/blob/main/ROADMAP.md): what is shipped, what is next, and the work that belongs to no single crate.

Per-crate detail stays with each crate: see its `CHANGELOG.md` for what has shipped and its own `ROADMAP.md` for what is planned.

## The names

Two of the crates are named for Plato's two kinds of memory, which is not decoration: it is what they do.

**ἀνάμνησις** (*anamnesis*), recollection, is Plato's term for recovering knowledge that was there all along. The crate takes a quantized checkpoint and recovers the precision that was compressed out of it.

**ὑπόμνησις** (*hypomnesis*), the external reminder that Socrates distrusts in the *Phaedrus*, is memory written down outside the mind. The crate measures a process's memory from outside that process.

## Development

Every crate here is developed with [Claude Code](https://claude.com/product/claude-code), with the git workflow managed in [Fork](https://fork.dev/), and follows a `CONVENTIONS.md` derived from [Amphigraphic-Strict](https://github.com/PCfVW/Amphigraphic-Strict)'s [Grit](https://github.com/PCfVW/Amphigraphic-Strict/tree/master/Grit), a deliberately strict Rust subset designed to raise AI coding accuracy.

anamnesis is benchmarked on [CodSpeed](https://codspeed.io/) (Linux aarch64) once per release, as a drift watch on its dequantisation hot path, where a performance regression could otherwise land without anybody noticing by eye. candle-fused-attn's kernels need a GPU that CI runners do not have, so they are measured before each release on five cards (an RTX 5060 Ti locally; an RTX 3090, 4090 and 5090 and an A100 on rented machines), and every number is kept in its [RESULTS.md](https://github.com/mi-for-the-rust-of-us/candle-fused-attn/blob/main/RESULTS.md).

Issues and pull requests are welcome on any of the five repositories.
