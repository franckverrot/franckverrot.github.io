---
title: "Building `lev`: a Jev-clone on LFM2.5-350M"
date: 2026-09-20 21:41:01
slug: "building-lev-on-lfm-2-5-350m"
tags:
  - ai
  - performance
  - open-source
  - jev
  - lfm
---

You've probably all seen TypeSafe's [Jev](https://docs.typesafe.ai/api), a model that answers 3 types of questions (choice, score, noul) about a document with probabilities.  It's extremely fast and the community has been wondering how it's made.

Luckily, Archer Hume seems to have worked out how `Jev` is most likely built and published his thoughts in [Jev's Architecture Unmasked](https://archerhume.com/posts/jevs-architecture-unmasked).  Jared Palmer turned that reconstruction into [kev](https://github.com/jaredpalmer/kev), an open implementation on Qwen that speaks the same API and ships with evals and ways to interact with it.  This weekend I cloned `kev` onto a different backbone, LiquidAI's [LFM2.5-350M](https://huggingface.co/LiquidAI/LFM2.5-350M), and called it [`lev`](https://github.com/franckverrot/lev) (still not great at naming things.)  

I picked a Liquid model like `LFM2.5-350M` mostly because I've been using it for a lot of tasks lately (and you'll find more on this blog,) and it runs quite efficiently on mobile.

The idea behind porting this to LFM is that I wanted to know whether `kev`'s recipe survived a new backbone that is pretty different from other models, and is also smaller in terms of parameters than competing models.


## The recipe

From a user perspective there's nothing really new `Jev` does, it's just a quite fast and simple API, and that's why it became so popular.  Whether we were able to do the same thing with `BERT` in 2018, or whether [it existed a year ago already](https://laya.convaiinnovations.com) are probably the right questions to ask...  I haven't been in the field long enough nor have researched it long enough to have a strong perspective here.  In any case, if you'd like to dive into details, they're in the [repo.](https://github.com/franckverrot/lev)  At a high level, we're training:

- LoRA adapters on every projection and on all 16 layers
- The pointer head (two small linear maps, and these ones are trained from scratch)

And the backbone and token embeddings stay frozen.

![How one request goes through `lev`](https://raw.githubusercontent.com/franckverrot/lev/master/docs/lev-architecture.png)

I gave it a shot pre and post training, zero-shotting it gave:

|         | Pre   | Post  |
| ------- | ----- | ----- |
| AG News | 0.62  | 0.88  |
| MNLI    | 0.43  | 0.75  |

Both datasets are in the training mix, so this is trained against untrained and nothing more.


## Performance

I've mostly reused `kev`'s training recipe and seed.  


|                                                      | lev-350m | kev-0.6b | Jev    |
| ---------------------------------------------------- | -------- | -------- | ------ |
| parameters                                           | 361M     | 607M     | hosted |
| accuracy, in-distribution                            | 0.773    | 0.805    | 0.845  |
| accuracy, out-of-domain                              | 0.546    | 0.598    | 0.857  |
| request with 3 questions                             | 25 ms    | 46 ms    |        |
| 24 questions on one state                            | 110 ms   | 223 ms   |        |
| weights in memory                                    | 1.44 GB  | 2.43 GB  |        |
| training, seconds per record                         | 0.25     | 0.44     |        |

(The Jev column comes from `kev`'s measurements of the hosted API.)

On accuracy `lev` is behind `kev-0.6b`, but `lev` wins on speed and memory.  The LFM tokenizer also needs about 14% more tokens than Qwen's for the same text, which takes back part of the gain...


### Is it any good?

It's less accurate than `kev-0.6b`, but in the same band as the other sub-1B entries on [JevBench](https://github.com/fstandhartinger/jevbench).  Regarding size and speed, it can answer a three-question request in 25 ms on a M2 Max 96GB, with 1.4 GB of weights in memory, and training takes about 40 minutes.

|                     | parameters | Intelligence | Calibration | JevBench  |
| ------------------- | ---------- | ------------ | ----------- | --------- |
| Jev 1.13.0          | hosted     | 90           | 83          | 75.4      |
| openJev Verdict 1.4 |            | 58           | 74          | 72.5      |
| Laya                | 421M       | 63           | 62          | 70.1      |
| jeff                | 400M       | 64           | 65          | 66.9      |
| kev 0.6B            | 607M       | 67           | 51          | 66.7      |
| **lev-350m**        | 361M       | 60           | 63          | ~68 (est.) |

(The `lev` row is my own run on the 231 public decisions of JevBench. I didn't submit anything, so I might be slightly off (and only slightly...))

To me the important part was that `kev`'s recipe carried over to a different type of language model architecture.  (One change was needed though: LFM's convolution layers ignore the attention mask, so I had to make them respect question boundaries.)


## To be tried next

Probably a bigger backbone like LFM2.5-1.2B, and ideally on a bigger GPU.  Hopefully I'll get a beefier machine soon, but if anyone wants to try this out on their GPU-rich infra, please do and report back :-)


## Some links

- Code: [github.com/franckverrot/lev](https://github.com/franckverrot/lev)
- Weights: [franckverrot/lev-350m](https://huggingface.co/franckverrot/lev-350m)
- Demo: [huggingface.co/spaces/franckverrot/lev](https://huggingface.co/spaces/franckverrot/lev)
- [kev](https://github.com/jaredpalmer/kev) by Jared Palmer
- [Jev's Architecture Unmasked](https://archerhume.com/posts/jevs-architecture-unmasked) by Archer Hume
- [TypeSafe System One API](https://docs.typesafe.ai/api)
- [LFM2 technical report](https://arxiv.org/abs/2511.23404)