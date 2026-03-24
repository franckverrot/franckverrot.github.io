---
title: "Running a 0.8B Model on an iPhone to Help My Kid Pick a College"
date: 2026-03-23 12:00:00
slug: "running-a-0-8b-model-on-an-iphone-to-help-my-kid-pick-a-college"
tags:
  - ai
  - ios
---

College search is a 10-dimensional optimization problem that families solve with spreadsheets, gut feelings, and way too many browser tabs.

My kid wants to study biology, play D1 soccer, and ideally not pay more than $40K/year.  Oh, and not biomedical, just bio.  That's four constraints already.  Add in state preferences, school size, acceptance rates, graduation rates, application systems (Common App?  ApplyTexas?  UC?), and now you're juggling variables that don't fit in your head.

<!--more-->

Parents and kids approach this differently.  Parents look at tuition, outcomes, debt loads.  Kids look at the campus, the team, the vibe ("I don't think I'm a UCSB type of guy, dad.")  You end up with dueling spreadsheets, shortlists of shortlists, and conversations that go in circles because nobody's looking at the same filtered view of the same data.

I wanted to build something we could both use.  An app where we each star schools independently (so you can see who picked what), with a search that understands plain English like "bio + d1 soccer under 40k".  No forms, no dropdowns for the initial search.  Just type what you want.

## What I built

<video autoplay loop muted playsinline style="max-width: 340px; display: block; margin: 0 auto 1.5em;">
  <source src="/images/send-this-kid-to-college-demo.mp4" type="video/mp4">
</video>

A SwiftUI iOS app called SendThisKidToCollege.  It bundles data from the [College Scorecard](https://collegescorecard.ed.gov/data/) (2,236 universities with tuition, acceptance rates, graduation rates, enrollment, median earnings, debt) and [EADA](https://ope.ed.gov/athletics/) athletics data (17 sports across NCAA D1/D2/D3, NAIA, NJCAA with participation counts and operating expenses.)  Plus bachelor's degree programs from the Field of Study dataset.

The app has three tabs:

- **Explore**: card grid with filters, sort, swipe-to-star/hide
- **AI Search**: conversational query builder powered by an on-device LLM
- **Shared Lists**: family sharing via Cloudflare Workers (parent and kid each see who starred what.)

The AI Search is the interesting part.  You type "bio + d1 soccer under 40k" and an on-device model parses that into structured filters: `{majors: ["Biology"], sport: "Soccer", division: "NCAA Division I", tuition_max: 40000}`.  Those filters get applied to the bundled data.  No server, no API calls, no data leaving the phone.

The question was: which model can actually do this on a phone?

## Searching for a model

I needed a model that:

1. Fits in iPhone memory (~1-2GB max for the model.)
2. Produces valid JSON with specific field names and values.
3. Only outputs fields the user asked for (no hallucinating extra filters.)
4. Gets the values right (state codes as "CA" not "California", acceptance rates as 0.3 not 30%.)

I built an eval harness with 30 test cases.  Each test case has a natural language query and expected JSON output.  Three metrics:

- **Field Precision**: of the fields the model outputs, what % were actually expected?  (Catches hallucinated extras.)
- **Field Recall**: of the expected fields, what % did the model include?  (Catches missing fields.)
- **Value Accuracy**: for correct fields, are the values right?

Plus a parse error count (model didn't produce valid JSON at all.)

### The results


| Model                                                                                | Size | Precision | Recall | Value Acc | Parse Errors |
| ------------------------------------------------------------------------------------ | ---- | --------- | ------ | --------- | ------------ |
| **[Qwen3.5-0.8B](https://huggingface.co/mlx-community/Qwen3.5-0.8B-MLX-4bit) (VLM)** | 0.8B | 9.3%      | 61.7%  | 38.3%     | 10/30        |
| Phi-4 Mini Instruct                                                                  | 3.8B | 6.8%      | 26.7%  | 21.7%     | 19/30        |
| LFM2.5 1.2B Instruct                                                                 | 1.2B | 5.7%      | 25.0%  | 5.8%      | 20/30        |
| Gemma3 1B it                                                                         | 1B   | 0.7%      | 3.3%   | 3.3%      | 28/30        |
| Qwen3 0.6B                                                                           | 0.6B | 0%        | 0%     | 0%        | 30/30        |


Nobody is good at this.  But Qwen3.5 is dramatically less bad than everything else.

## Some surprises

Phi-4 at 3.8B scored worse than Qwen3.5 at 0.8B.  Nearly 5x the parameters, worse at structured output.  19 parse errors vs 10.  Lower recall (27% vs 62%.)  Model size is not the determining factor here.

Qwen3 0.6B and Gemma3 1B produced zero and near-zero valid outputs.  These models simply cannot follow a structured output prompt at this scale.  They ramble, repeat the prompt back, or produce text that isn't JSON.

Qwen3.5 is technically a vision-language model (VLM.)  I'm using it for text-only input.  The VLM architecture seems to handle structured output better than pure text LLMs at sub-1B scale.  I don't fully understand why.  It might be that the multi-modal training gives the model a better grasp of format and structure, or it might just be that Qwen3.5 is a newer architecture with better training data.

The 9.3% precision is the real problem, not recall.  The model finds the right fields 62% of the time, but buries them under 10+ hallucinated extras.  For "soccer schools", it outputs the correct `sport: "Soccer"` but also throws in `type: "Public"`, `enrollment_min: 10000`, `graduation_min: 0.7`, and a bunch of other fields nobody asked for.

## Attempt at fine-tuning

I tried LoRA fine-tuning on Qwen3.5-0.8B using [mlx-vlm](https://github.com/Blaizzy/mlx-vlm) with 390 training examples.  This went through several iterations.

**Attempt 1: `mlx-vlm` default LoRA (all linear layers.)**  Adapted 372 tensors across every linear layer in the model.  With only 125 training examples in the first pass, the model catastrophically overfitted.  Training loss dropped to 0.00003 in 125 steps.  The model looped on fragments of the system prompt instead of producing JSON.  Completely destroyed.

**Attempt 2: restricted LoRA (`q_proj` + `v_proj` only.)**  I wrote a custom training script to only adapt the attention Q and V projections (24 tensors instead of 372), matching the approach used in a working fine-tune of Qwen3 for another project.  Same catastrophic overfitting, just slightly different garbage output.

**Attempt 3: lower learning rate.**  Tried `lr` from `1e-5` down to `5e-7`.  At `1e-5` the model memorizes and breaks.  At `5e-7` the adapter barely changes anything.  At `5e-6` I got marginally better results (9.7% precision vs 9.3% baseline) but still 15/30 parse errors.  Not meaningful improvement.

**Attempt 4: more data (390 examples.)**  Generated 242 augmented examples covering every sport, major, state, division, and their combinations.  Total 390 training examples.  Same results.  The model goes from underfitting to overfitting within about 15 training steps.  There's no stable zone.

Why fine-tuning failed (so far): the examples are tiny.  Each training example is a query + a short JSON output, maybe 33 tokens.  Compare to a conversational fine-tune where each example might be 200+ tokens.  The model doesn't get enough signal per example to learn the pattern without memorizing the data.  I'd likely need 1,000+ examples or much longer training sequences with more context.

The [MLX](https://github.com/ml-explore/mlx) ecosystem deserves a shoutout here.  [Prince Canuma](https://github.com/Blaizzy) maintains `mlx-vlm` and has made running vision-language models on Apple Silicon genuinely accessible.  The broader MLX team at Apple and the community around `mlx-lm`, `mlx-vlm`, and the `mlx-community` model hub on HuggingFace are doing incredible work.  The fact that I can fine-tune a VLM on my laptop, convert a PyTorch classifier to CoreML in a few lines, and run inference on a phone is a testament to what these folks have built.  My fine-tuning didn't work for this particular task, but the tooling itself is solid and I'll be back.  Folks, if you read this: lunch on me the day we meet :-)

## The eval infrastructure

One thing that worked well: the eval setup.  I use [LangSmith](https://www.langchain.com/langsmith/observability) for tracking results and a Python script that loads any model (VLM or text LLM) and runs it against the test cases.

```bash
# Eval any model
make eval-model MODEL=mlx-community/Qwen3.5-0.8B-4bit BACKEND=vlm NAME=qwen35
make eval-model MODEL=mlx-community/Phi-4-mini-instruct-4bit BACKEND=lm NAME=phi4
make eval-model MODEL=mlx-community/gemma-3-1b-it-4bit BACKEND=lm NAME=gemma3

# Compare all results
make eval-compare
```

The `--backend` flag switches between `mlx-vlm` (for VLMs) and `mlx-lm` (for text LLMs.)  This lets you throw any model at the same test suite.  All results go to LangSmith so you can compare runs over time.

Scoring is strict: I check field names match exactly, values match exactly, and penalize any field the model outputs that wasn't in the expected output.  No partial credit.

## Rethinking the approach: classification, not generation

After staring at the eval results, I realized something.  The model finds the right fields 62% of the time.  It buries them under hallucinated garbage.  For "soccer schools", it outputs `sport: "Soccer"` (correct!) alongside `state: "CA"`, `type: "Public"`, `enrollment_min: 10000`, and a dozen other fields nobody asked for.

The model understands the query.  It just can't stop generating.

I was asking the model to do five things at once: understand intent, map to field names, map to exact enum values, decide what to omit, and produce syntactically valid JSON.  At 0.8B parameters, that's too much.  What I actually needed was classification.  "Does this query mention a sport?  Which one?"  That's a 19-class classification problem (18 sports + none.)  "Which state?"  59-class multi-label.  "Which major?"  45-class multi-label.  Every filter dimension is a bounded classification task with a known label set.

### The multi-head classifier

I replaced the LLM with a [DistilBERT](https://huggingface.co/distilbert/distilbert-base-uncased) encoder (66M parameters) and 14 classification heads, one per filter dimension:

```
  "bio + d1 soccer under 40k"
              |
     +--------+--------+
     |  Regex / Rules   |  <-- extracts explicit numbers
     +--------+--------+
     +--------+--------+
     |   DistilBERT    |  <-- shared encoder (one forward pass)
     +--------+--------+
              | [CLS] embedding
    +---+---+-+--+---+---+---+---+---+---+---+---+---+---+
    v   v   v    v   v   v   v   v   v   v   v   v   v   v
   14 classification heads (sport, division, state, major, ...)
```

Each head is a single `Linear(768, N)` layer that can only output from its defined label set.  The sport head picks from 18 sports or "none."  It literally cannot hallucinate a state code, that's a different head's job.

A deterministic regex layer runs first and extracts explicit numbers ("under 40k" becomes `tuition_max: 40000`.)  The classifier handles the semantic understanding: fuzzy terms like "cheap," "selective," "east coast."


| Head            | Type         | Classes | Examples                     |
| --------------- | ------------ | ------- | ---------------------------- |
| sport           | single-label | 19      | Which sport (18 + none)      |
| sport_gender    | single-label | 3       | men's / women's / both       |
| division        | single-label | 6       | D1/D2/D3/NAIA/NJCAA/none     |
| state           | multi-label  | 59      | US states and territories    |
| country         | multi-label  | 19      | Country codes                |
| type            | single-label | 4       | Public / private / none      |
| system          | multi-label  | 5       | Common App, UC, etc.         |
| major (include) | multi-label  | 45      | Major keywords               |
| major (exclude) | multi-label  | 45      | "not biomed"                 |
| search intent   | binary       | 2       | "find me Stanford" vs filter |
| tuition         | single-label | 6       | cheap/affordable/expensive   |
| enrollment      | single-label | 5       | small/mid/large              |
| selectivity     | single-label | 5       | selective/very selective     |
| graduation      | single-label | 3       | high/very high               |


Total: 226 output neurons.  The heads add 173K parameters on top of the 66M backbone.  With INT8 quantization the model is 63.5MB, 87% smaller than the 500MB Qwen3.5 VLM, and inference is a single forward pass (~2ms) instead of autoregressive token generation (~2-5 seconds.)

### Training results

I started with the same 390 examples that failed at fine-tuning the LLM.  Then I systematically augmented: every sport x 6 phrasings, every major x 7 phrasings, every state x 6 phrasings, regional mappings ("east coast" becomes the right 11 states), negation patterns, fuzzy qualifiers, multi-constraint combinations.  3,320 examples total across 7 rounds.


| Round           | Examples | Precision  | Recall    | Value Acc | Parse Errors |
| --------------- | -------- | ---------- | --------- | --------- | ------------ |
| LLM baseline    | --       | 9.3%       | 61.7%     | 38.3%     | 10/30        |
| LLM fine-tuned  | 390      | 9.7%       | 43.1%     | 21.7%     | 15/30        |
| R1 (classifier) | 390      | 60.0%      | 52.2%     | 48.9%     | **0/30**     |
| R2              | 3,127    | 96.7%      | 80.8%     | 72.5%     | **0/30**     |
| R3              | 3,228    | **100.0%** | 87.8%     | 84.4%     | **0/30**     |
| R6              | 3,437    | **100.0%** | 97.2%     | 93.9%     | **0/30**     |
| R7 (final)      | 3,320    | **100.0%** | **95.3%** | **91.9%** | **0/30**     |


From 9% precision to 100%.  From 10 parse errors to zero.  From 38% value accuracy to 92%.  The model is smaller, faster, and dramatically more accurate.

Parse errors are zero by construction: there's no JSON generation, no text to parse.  Every output is assembled from fixed label predictions.  The classifier either gets it right or misses a field; it never produces garbage.

R6 had higher recall (97.2%) from aggressive oversampling of targeted patterns.  R7 traded a bit of that for much better generalization: 125 examples written as real users would type, things like "I want to study biology and play soccer", "anywhere but California", "somewhere warm with good CS", "I can't afford more than 40 thousand."  The starter queries the app shows to users are now all heavily trained.

The remaining 4 failures (out of 30) all have 100% precision, the classifier just misses one head in complex multi-field queries.  The `exclude_majors` head and `state` head are the weakest at co-activating alongside other filters.

**EDIT: my friend Mark reached out on LinkedIn and pointed out something I should have seen earlier.  He said the co-activation problem (heads not firing together) was probably a training data issue: if heads mostly appear in isolation during training, the model learns to fire them independently.  He suggested generating combinatorial examples that hit 4-5 heads at once.**

**He was completely right.  R7 had 4 remaining failures where the model wouldn't fire multiple heads simultaneously -- "d3 lacrosse in massachusetts" got lacrosse and D3 but missed the state.  I generated 119 examples that force 3-5 heads to fire at once, written in both shorthand ("D1 soccer + bio + California + under 40k") and conversational style ("I want to play D1 soccer, study biology, preferably in California, and not break the bank".)  R8: 0 failures.  100% precision, 100% recall.  Thanks Mark.**

### The mobile integration

The classifier runs on-device via [CoreML](https://developer.apple.com/documentation/coreml).  The iOS app has an A/B toggle (`@AppStorage("aiSearchEngine")`) to switch between the classifier and the original MLX LLM.  The classifier path:

1. User types query.
2. DistilBERT WordPiece tokenizer encodes to token IDs (Swift, ~1ms.)
3. Regex preprocessing extracts explicit numbers ("under 40k" becomes `tuition_max: 40000`.)
4. CoreML runs the 14-head classifier in a single forward pass (~15ms.)
5. Assembly layer maps head predictions + regex values to structured filter.
6. Filter applied to 2,236 universities.

No download.  No waiting.  Type and see results.

### Quantization: how small can we go?

CoreML supports post-training weight quantization, no retraining needed, just a few lines after conversion.


| Variant | Size   | Perfect Matches | Notes                |
| ------- | ------ | --------------- | -------------------- |
| FP16    | 126 MB | 26/30           | Baseline             |
| INT8    | 64 MB  | ~26/30          | Essentially lossless |
| INT4    | 32 MB  | 20/30           | 6 predictions flip   |


INT8 is the sweet spot: half the size, no accuracy loss.  The argmax and sigmoid threshold decisions that classifiers make are robust to 8-bit weight precision.  INT4 starts losing predictions on the multi-label heads (`state`, `major`) where sigmoid values near 0.5 flip differently at 4-bit precision.

For context: the original Qwen3.5 VLM was ~500MB and scored 9% precision.  The INT8 classifier is 64MB and scores 100% precision.  8x smaller, 11x more precise.

### Why not a smaller backbone?

I tested `all-MiniLM-L6-v2`, a 22M parameter sentence transformer, 3x smaller than DistilBERT.  On paper it should be better: trained on 1 billion sentence pairs, specifically optimized for short text embeddings.


| Backbone   | Params | Precision  | Recall    | Size (FP16) |
| ---------- | ------ | ---------- | --------- | ----------- |
| DistilBERT | 66M    | **100.0%** | **95.3%** | 126 MB      |
| MiniLM     | 22M    | 93.3%      | 80.8%     | 44 MB       |


MiniLM's 384-dimensional embedding doesn't have enough capacity for 14 heads.  The multi-label heads suffer most: state accuracy plateaus at 68.7% (vs 74.3% with DistilBERT) regardless of how many epochs you train.  The model runs out of dimensions to encode all 14 filter concepts simultaneously.

The INT8 quantized DistilBERT at 64MB is only 20MB larger than FP16 MiniLM at 44MB.  That 20MB buys 100% precision vs 93%, and 95% recall vs 81%.  Easy choice.

### Why not CreateML?

Apple's CreateML has a Text Classification template that trains a classifier from labeled text in minutes, right in Xcode.  I considered it.

The problem is that this task isn't one classification, it's 14 simultaneous classifications, several of which are multi-label (`state` can be `["NY", "MA", "CT"]` for "new england schools".)  CreateML's Text Classification is single-label: one input, one output.  You'd need 14 separate models, 14 inference calls, and no shared understanding between them.

Worse, CreateML uses bag-of-words or shallow transfer learning under the hood.  It doesn't use a transformer.  It doesn't understand word order.  "Schools outside Texas" and "schools in Texas" would look similar to it, both contain "schools" and "Texas."  The DistilBERT backbone understands the negation because the attention mechanism captures the relationship between "outside" and "Texas."

CreateML would win on size (1-5MB vs 64MB) and simplicity (drag CSV, click train.)  But it would lose on accuracy for every query that requires semantic understanding: "somewhere warm," "I can't afford more than 40k," "east coast with good academics."  Those need a model that understands language, not one that counts words.

## Next steps

1. More real-user training data.  Get that sweet data from LangSmith and shove it into our training pipelines.
2. Add more heads.  The architecture scales by adding heads, not by rewriting prompts.  "Schools near mountains" or "good campus food" would be new heads.
3. Send that kid to college... that was the idea, right?

## The tech stack

- SwiftUI, SwiftData, iOS 26
- DistilBERT multi-head classifier (14 heads, 66.5M params, 63.5MB INT8 quantized) for query parsing via CoreML
- Regex preprocessing for explicit number extraction
- 2,236 universities from College Scorecard + EADA athletics data, bundled in-app (3.5MB JSON)
- Cloudflare Workers + KV for shared list sync
- Python eval harness with LangSmith integration
- PyTorch + HuggingFace Transformers for classifier training

The whole thing runs on a phone.  No backend except the optional shared lists sync.  The university data, the model, the filtering, the starring, all local.

## Takeaway

If you're trying to run structured output tasks on a phone, don't reach for a generative model.  I spent a bunch of time trying to make a 0.8B LLM produce reliable JSON and it just can't do it.  9% precision, 10 parse errors out of 30 queries.

Once I reframed the problem as classification instead of generation, everything fell into place.  One shared encoder, 14 small heads, each constrained to its label set:
- no hallucinations
- parsing worked 100%
- love the determinism and debuggability
- really fast

With some tradeoffs when it comes to UX, a 66M simple model is doing (almost) what a 0.8B language model couldn't.

The college search problem is real, I'm only scratching the surface.  Billions (again, almost) schools, thousands of sports, hundreds of majors, 50 states (not accounting for "Can I go study in Japan?"), and every family has different priorities.  An app that lets you type what you want and instantly see what matches, with your kid starring their picks and you starring yours... that was worth the rabbit hole.

What I like about this investigation is having spent brain time on UX too.  Most websites would benefit from a search that isn't passive (regardless of how clever the result set will be,) but would tweak available search facets/criteria for the user to then refine manually, or just see which ones have been turned on/off / configured.