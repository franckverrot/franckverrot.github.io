---
title: "Trying to beat MLX for LFM2.5-350M"
date: 2026-09-26 14:12:42
slug: "trying-to-beat-mlx-for-lfm2-5-350m"
tags:
  - ai
  - performance
  - ios
  - open-source
---

I've blogged a few times about small language models running locally, [how to run one on a phone](/blog/2026/03/23/running-a-0-8b-model-on-an-iphone-to-help-my-kid-pick-a-college/) and [how to fine-tune one](/blog/2026/03/31/fine-tuning-actually-worked/), and it left me wondering (it's been a couple of months in the making, I got busy with other things in the mean time...) what it would take to run inference on an Apple GPU faster than the go-to frameworks.  So I wrote an inference engine for a model I really like ([LFM2.5-350M](https://huggingface.co/LiquidAI/LFM2.5-350M),) and measured it against [llama.cpp](https://github.com/ggml-org/llama.cpp) and [MLX](https://github.com/ml-explore/mlx) on the Mac, and then against MLX Swift on my phone.

I could have named this post "How to cut inference time and memory with MLX by 50%" to make the internet want to read it... and maybe I should have.  Anyways: after a lot of trial and error, it eventually worked.

* On my Mac: it decodes faster than both baselines at a fraction of the memory.
* On my iPhone 17 Pro Max: it's ahead of MLX Swift on time to first token at every prompt length I measured, by 1.7x to 4x, with the replies identical word for word.

I think model-specific **and** architecture-specific inference engines are where edge AI is going.  General frameworks have many useful abstractions that shouldn't be shipped on devices.

Oh, and I named the engine `grouille`, French colloquialism for "hurry up."  Decided to name it this way after verbalizing "aller grouille j'dois y aller" to Claude a minute before starting this experiment...

<!--more-->

## The bet

`grouille` can't and won't do magic.  General frameworks, highly reviewed by people smarter than me, could implement the same thing.  There's no new research in here either: flash-decoding attention, fused kernels, quantization scales, all of those are known techniques.  Where I thought I could win is by stripping the engine from every capability I wasn't using.  MLX, llama.cpp and others have to stay generalists and serve more than one model; `grouille` runs exactly one model, on the platforms I own, and pretty much nothing else.  The wins below are a direct consequence of giving up the flexibility you're used to.

In the end, I built a Rust engine over Metal, and around 2k lines of MSL ([Metal Shading Language](https://developer.apple.com/metal/resources/)) for the 30+ kernels needed (and weight files optimized for how the kernels want the data laid out.)  Here's how a token goes through the model:

![Simple diagram showing decoding for a single token in `grouille`.  80 dispatches, with the CPU never waiting, and memory being allocated at startup.](/images/grouille/decode-token.svg)

Two things to note: memory is allocated at startup, so it doesn't shoot up during decoding (something I wanted for mobile,) and the small operations are embedded in the kernels, so they're not in the loop at all.  I matched the reference implementation first for correctness, then attacked speed.  On the Mac:

| quant | engine | decode tok/s (128-token prompt) | TTFT ms (128-token prompt) | prefill tok/s (2048-token prompt) | peak MB |
| --- | --- | --- | --- | --- | --- |
| q4 | **grouille** | **915.3** | **13.4** | 10,345 | **136** |
| q4 | llama.cpp | 580.5 | 27.7 | **12,565** | 1,889 |
| q4 | mlx-lm | 702.5 | 25.8 | 12,254 | 476 |
| q8 | **grouille** | **641.0** | **13.6** | 10,296 | **134** |
| q8 | llama.cpp | 465.1 | 26.0 | **14,152** | 2,039 |
| q8 | mlx-lm | 526.3 | 28.2 | 11,736 | 653 |
| f16 | **grouille** | **417.8** | **13.3** | 10,578 | **139** |
| f16 | llama.cpp | 335.9 | 26.8 | **15,020** | 2,372 |

![grouille ended up ahead on decode, time to first token and memory, but behind on the long 2048-token prefill...](/images/grouille/mac-scoreboard.svg)

`grouille` leads on decode at native precision (fp16) and at every quantization I've tried, but it loses on long prompts, especially at q8.

## How prefill works

![One chat turn: the prompt's 17 tokens go through the model together in prefill, then decode produces the reply one token at a time](/images/grouille/chat-turn.svg)

Prefill and decode hit two different limits:
1. Prefill is limited by compute: every weight is multiplied with every prompt token.
2. Decode is limited by memory speed: the model streams its 204 MB of (4-bit) weights for each and every token.  At the phone's 60 to 70 GB/s, that's about 3 ms per token.

This post is mostly about prefill because what matters was how well we're leveraging the GPU for matrix operations.

![Architecture of LFM2.5-350M, because it looks cool.](/images/grouille/lfm25-architecture.svg)

The wide boxes are projections: a token's vector multiplied by a weight matrix.  Every block runs four of them in prefill, so a prompt goes through 64 matrix multiplies before the first token can be sampled.  For a 17-token prompt every one of them is tiny (17 vectors of 1024 numbers times a weight matrix,) and a GPU is built for beefier workloads.  So the question is how much each one costs before it does any useful work, because we pay that 64 times.

Quick detour on how a GPU multiplies.  It computes the output in squares of 64x64 numbers called tiles.  One tile is the work of one threadgroup (128 or 256 threads sharing a small fast memory,) and launching all the tiles of one multiply together is a dispatch.

Here's the `out` projection:

![A projection under a microscope](/images/grouille/matmul-tiles.svg)

Two things to notice:
1. Our 17-token prompt fills 17 rows of one row of tiles, and **still pays for all 64.**
2. The width of a projection sets how many tiles there are: `gate` and `up` produce 9216 numbers per token, so 144 tiles across; `out` and `down` produce 1024, so 16.  **Narrow projections give the GPU very little to do at once**, and that gets worse on a phone: the iPhone 17 Pro Max has 6 GPU cores, the M2 Max has 38.

## On the phone

I built a quick SwiftUI app to compare `grouille` and [`mlx-swift-lm`](https://github.com/ml-explore/mlx-swift-lm) on the same prompts:

| | grouille | MLX Swift |
| --- | --- | --- |
| engine load, warm | ~0.07 s | 0.18 s |
| memory after three turns | ~160 MB | ~570 MB |
| time to first token | ~70 ms | ~40 ms |
| decode (~70 tokens) | ~250 tok/s | ~250 tok/s |

Decode was on par (as we could have predicted,) and MLX won on TTFT.  `grouille` loaded 3x faster though, at a quarter of the memory, and stayed flat across turns (MLX grows about 100 MB per turn and reached 1.2 GB after a ~600-token prompt.)

I closed the lid that night thinking the phone was just worse at keeping its 6 cores busy.  I hadn't profiled anything, it was a guess from looking at two curves.


## Figuring out some fixes

Half the fixed cost turned out to be threadgroups with nothing to do.  Every prefill kernel launched a grid for a full 512-row chunk whatever the prompt held, so at 17 tokens 7 of the 8 rows of threadgroups read one word and exited.  They still have to be scheduled, and on 6 cores that cost as much as the real tiles.  In the end, launching only the threadgroups the prompt needs halved prefill on a 17-token prompt on the phone, and took a third off on the Mac.

![The threadgroups one out projection launches for a 17-token prompt, before and after sizing the grid by the prompt: 7 of 8 rows of them had nothing to do](/images/grouille/grids-by-chunk.svg)

With the empty threadgroups gone, what was left was the tile itself: about 20 ms per 64-row tile on this phone.

Turns out the phone's A19 Pro has neural accelerators and I wasn't leveraging them.  I got to find this out by [digging into MLX's source.](https://github.com/ml-explore/mlx/blob/v0.31.1/mlx/backend/metal/device.h#L268)  These neural accelerators are matrix-multiply units sitting next to the regular shader units, and [Metal 4](https://developer.apple.com/videos/play/wwdc2025/205/) lets you call it from a shader with a single operation, [`matmul2d`](https://developer.apple.com/download/files/Metal-Performance-Primitives-Programming-Guide.pdf) (on GPUs without accelerators, it still runs, on the regular GPU units.)

After updating my tile kernel to call `matmul2d` I got some interesting results:

![Engine-side prefill on the iPhone 17 Pro Max for four prompt lengths: grouille before, with the grids sized by the prompt, on the tensor ops, and MLX Swift](/images/grouille/phone-prefill.svg)

The 616-token prompt runs 2.5x faster than MLX Swift, and the first token comes out 1.7x to 4x sooner than MLX (identical output.)

I tried a few other things and they didn't work, but for the ones that worked none of those were ground breaking AI research.  So essentially, learning how to use the hardware properly and avoiding reprocessing data were winning tactics...

Lessons learned... learn the architecture 🤷


## It's open source

None of this would have been possible without a heavy dose of agentic AI: the docs are great, the math is approachable(-ish, could look daunting at times...) and a willingness to understand how all of this works under the hood is required, but without Claude and the likes it would have taken me months, so I'm glad these things exist.

Everything is in [the repo,](https://github.com/franckverrot/grouille) there's even some notes from my dabbling.