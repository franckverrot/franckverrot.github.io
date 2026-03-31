---
title: "Fine-Tuning Actually Worked"
date: 2026-03-31 12:00:00
slug: "fine-tuning-actually-worked"
tags:
  - ai
  - ios
---

Quick follow-up to the [college search post](/blog/2026/03/23/running-a-0-8b-model-on-an-iphone-to-help-my-kid-pick-a-college/).  Last time, I gave up on fine-tuning a generative model and built a DistilBERT classifier instead.  The classifier works great (100% precision, 100% recall), but I always had this nagging feeling that fine-tuning should have worked if I'd found the right model and the right tooling.

Today Liquid AI released [LFM2.5-350M](https://www.liquid.ai/blog/lfm2-5-350m-no-size-left-behind), and it changes the picture.

<!--more-->

## What happened

LFM2.5-350M is a 350M parameter model with a hybrid architecture.  I ran it through the same eval harness as everything else.  Base model: 30/30 parse errors.  Zero useful output.  Same story as Qwen3 0.6B and Gemma3 1B.

Then I fine-tuned it with LoRA using `mlx-lm` and the same 390 training examples that failed on Qwen3.5-0.8B.

| Model | Size | Precision | Recall | Value Acc | Parse Errors |
| --- | --- | --- | --- | --- | --- |
| **LFM2.5-350M + LoRA** | 0.35B | 76.9% | 78.9% | 85.0% | **0/30** |
| Qwen3.5-0.8B VLM (base) | 0.8B | 9.3% | 61.7% | 38.3% | 10/30 |
| Phi-4 Mini Instruct | 3.8B | 6.8% | 26.7% | 21.7% | 19/30 |
| LFM2.5 1.2B Instruct | 1.2B | 5.7% | 25.0% | 5.8% | 20/30 |
| Gemma3 1B it | 1B | 0.7% | 3.3% | 3.3% | 28/30 |
| LFM2.5-350M (base) | 0.35B | 0% | 0% | 0% | 30/30 |
| Qwen3 0.6B | 0.6B | 0% | 0% | 0% | 30/30 |

Zero parse errors.  77% precision, 79% recall, 85% value accuracy.  From a model less than half the size of Qwen3.5.

For context: Phi-4 Mini at 3.8B (nearly 11x the parameters) scored worse on every metric.

## Why this one worked

Training took under 30 seconds on my Mac.  Validation loss dropped steadily with no catastrophic overfitting.  That's the stable training zone I never found with Qwen3.5.

Two things made the difference:

1. `mlx-lm` instead of `mlx-vlm`.  Full control over which layers to adapt and proper validation loss during training.
2. The model itself.  LFM2.5-350M didn't overfit despite a small dataset.  Same 390 examples that destroyed Qwen3.5 in 15 steps trained cleanly here.  I don't know if that's the architecture or the pre-training data, but something about this model is more amenable to small-data LoRA.

## From 77% to 93%

The first fine-tune landed at 77% precision / 79% recall / 85% value accuracy.  I looked at the 5 test cases scoring below 50% to understand what the model was getting wrong:

- `"find me stanford"` -- put Stanford in `system` instead of `search`.
- `"private schools over 50k tuition"` -- used `tuition_max` instead of `tuition_min`, put `type` in `system`.
- `"apply texas schools with bio"` -- output `state: ["TX"]` instead of `system: ["ApplyTexas"]`.
- `"large public d1 football"` -- missed `enrollment_min` and `sport`, put "Public" in `system`.
- `"good schools"` -- invented `majors: ["Good Schools"]` instead of returning empty.

The model was confusing three things: the `type` field vs the `system` field, `tuition_min` vs `tuition_max`, and the `search` field.

I added 85 targeted examples (475 total): search queries ("find Duke", "find Princeton"), type/system combos ("public schools with bio", "apply texas with nursing"), tuition direction patterns ("schools over 50k", "expensive above 60k"), and vague queries ("good programs", "nice schools") to reinforce empty output.

Training: 500 iterations instead of 200.  Validation loss kept dropping through iter 350 (0.048), then started rising at iter 400-500 (0.072.)  I picked the iter 350 checkpoint.

| | Precision | Recall | Value Acc | Parse Errors |
| --- | --- | --- | --- | --- |
| Run 1 (390 ex, iter 200) | 76.9% | 78.9% | 85.0% | 0/30 |
| Run 2 (475 ex, iter 350) | 92.8% | 92.8% | 93.9% | 0/30 |

All five previously failing test cases now pass.  16 point improvement across the board from 85 targeted examples and checkpoint selection.

## Where it stands

The fine-tuned LFM2.5-350M is ~175MB quantized (4-bit) and uses ~200MB RAM at inference.  Fits on any iPhone.

At 93% precision and recall, the gap with the DistilBERT classifier (100%/100%) is narrowing fast.  The classifier is still more accurate, smaller (64MB), and faster (~15ms vs a few seconds for autoregressive generation.)  But the fine-tuned model produces natural language JSON, handles queries the classifier has never seen in a more flexible way, and didn't require building 14 classification heads.  With another round or two of targeted data, I think this closes the gap entirely.

I'm leaning toward the fine-tuned model as the production path now.  The flexibility of a generative model that actually works is worth a small accuracy tradeoff, and the accuracy tradeoff keeps shrinking.

## Liquid AI cooked

I feel young phrasing it this way but I want to give credit where it's due.  The Liquid AI team shipped a 350M model with a hybrid architecture that, after 30 seconds of fine-tuning on a laptop, outperforms instruction-tuned models 3-10x its size on structured output.  Something about this model learns efficiently from small datasets in a way that the other models I tested couldn't.

I don't know if this generalizes beyond my use case.  But for on-device structured output from natural language, this is the most impressive result I've seen at this parameter count.
