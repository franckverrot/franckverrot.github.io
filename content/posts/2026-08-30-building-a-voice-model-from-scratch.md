---
title: "Building a voice model from scratch"
date: 2026-08-30 09:12:04
slug: "building-a-voice-model-from-scratch"
tags:
  - ai
  - voice-model
  - tts
  - text-to-speech
  - open-weight
---

Lately, I've spent some time building a new tiny voice model.  I had never built one, yet, and while I had some vague understanding about how they were designed, I've taken more time writing one from scratch and see how far I could go with minimal investment: the idea was to train it locally with my current setup (M2 Max, 96GB) and avoid spending more money renting GPUs (for now) or buying more expensive hardware.

Anyways, maybe this will be a series of posts, where I will dig into hand-written Metal kernels, writing a training toolchain from scratch (no PyTorch, no Python at all actually: Swift only), and finding optimizations to make it an acceptable voice model.

Heads-up: this won't compete with [Kokoro](https://huggingface.co/hexgrad/Kokoro-82M), [Matcha](https://github.com/shivammehta25/Matcha-TTS), or other smaller models, I just want to nail the theory and the practice.  I'm less interested in climbing at the top of a leaderboard than building solid foundations for a potentially larger model one day.  Here's a live example:

<audio controls preload="none" src="/images/vovo/vocoder-gan.m4a"></audio>

It's not studio quality (crackly sound), but for a 20M model, trained in about 13 minutes on a modest setup, it is definitely not a bad outcome.  The inference side is open source ([vovo-core](https://github.com/franckverrot/vovo-core)) and there's a [demo](https://huggingface.co/spaces/franckverrot/vovo) you can type into (slow on the free CPU it runs on, fast on a Mac.)

Let's dive into some more details now.

<!--more-->

## Why not reuse a ML framework

I have been using PyTorch and MLX for a while now.  I appreciate how fast, flexible, and understandable they are, yet I wish I had written one from scratch as well, so I dove into this rabbit hole with my agents to build the tensor library, the GPU kernels, the autodiff/autograd/optimizers myself, and I'll talk about it in another post.  Definitely not as fun as building something a bit more innovative, but building it was informative (and digging into why Muon was better than AdamW for what I was trying to accomplish is the type of data science and math that I like solving, so it did provide a nice shot of dopamine when I found a solution.)

I also don't intend to start a multi-day training run... and I'd rather rent a GPU elsewhere than babysit my Mac for so long.  The goal was really to just see how fast I could get that thing to talk to me, and then improve it over time.  That constraint informed a lot of other decisions, and motivated an architecture where getting stuff done was the priority.


## A voice model is not one network

I went full Jigoro Kano on this project and approached it with a beginner's mind: "it is not important to be better than someone else, but to be better than yesterday."  I had no idea what the backbone behind a voice model was... The first surprise was that I was expecting a single model end to end, but a voice model is more of a pipeline:
 
1. Text becomes sounds: "Hello, world!" becomes a sequence of tokens and phones: `h ə ˈ ɫ o ʊ , ˈ w ɝ ɫ d !`.  To be clear, a phone is a speech sound: `Hello` is one word, `h ə ˈ ɫ o ʊ` is 5 phones (`ˈ` is a token, but not a phone...)  The model never sees the spelling, it sees that sequence
2. Sounds become a picture: the target is a mel spectrogram (time on one axis, frequency bands on the other, loudness as color)
3. The model learns to paint that picture from the phones
4. Another model, the "vocoder," turns the picture back into a waveform

Most of the work was not spent on the neural network, as I expected, but it was spent on aligning letters and time, on the evaluation loop, and on building tooling to observe what the model was actually doing.


## Building a text front-end

Before the model sees the text, data needs to be normalized:
* `$12.50 at 4:05` needs to be pronounced "twelve dollars fifty cents at four oh five"
* years need to be read the way people read them ("nineteen eighty four" but "two thousand five")
* same thing with ordinals, decimals, etc.
* some abbreviations expand differently depending on the context.

Each word is looked up in a pronunciation dictionary, and I used [ipa-dict](https://github.com/open-dict-data/ipa-dict), which contains about 126,000 English entries.  Unknown words will fallback to another set of rules.

The output will be a sequence of tokens (including punctuation as it carries both pauses and intonation) from a fixed alphabet of 67 phone symbols.  Ideally I get to train with prosody tags as well in the future.

A small thing I did not really expect but ended up uncovering another unknown unknown: naming the model "Vovo".  As I built the evaluation toolchain with Apple's on-device speech recognizer (to transcribe what the model says,) it kept hearing "Vovo" as "Dolo"...  And Apple's ASR has never heard that name either.  To fix this, there's now a pronunciation override file, [and `vovo` is in it](https://github.com/franckverrot/vovo-mlx/blob/main/vovo_mlx/text/data/overrides.txt#L3).


## The mel spectrogram

"Mel" is short for melody, and its scale measures how high or low a sound *feels* to a human.  A mel spectrogram is a 2D picture of sound:
* time on the X-axis
* frequency bands on the Y-axis
* the brightness is for how loud the sound is

![A mel spectrogram viewed in the editor](/images/vovo/mel-editor-2d.jpg)

A normal spectrogram spaces out frequency evenly: a jump from 100Hz to 200Hz gets the same width as one from 8kHz to 8.1kHz.  To your ear (I'll assume you're human if you're reading this... and if you're not, you'll have to imagine,) the first jump is a full octave, but the second (8k to 8.1k) is barely noticeable.  The mel scale warps the axis to match that: tighter bands at the bottom where changes in pitch matter, and wider bands at the top where they don't.  It's the same thing for speech: most of what tells one phone from another is down in that low-to-mid range, so you want your bands there, not wasted up high (again, unless you're a dog or a cat).

To actually draw one of these pictures, you need to pick a few numbers, so here are mine:
* the X-axis is sliced into frames of about 10 milliseconds, so each column is a tiny slice of the recording
* the Y-axis gets 100 frequency bands, packed tighter at the bottom the way I just described
* the recording is 24,000 samples a second (that's the `24 kHz`)
* brightness is the log of how loud each band is in that frame (so a shout doesn't blow out the picture compared to a whisper)

There was no groundbreaking work here, every TTS pipeline out there already uses a config that's pretty similar (sampling and number of bands may differ a bit.)  And I reused [Vocos](https://github.com/gemelo-ai/vocos) to synthesize the audio, no new work here.

One thing that bit me is that these params have to be matching (with some margin of error...) everywhere in the pipeline: the data prep, the model's output, and the vocoder all need to align on those.  If anything drifts, you would still get sound out the other end, but it wouldn't be speech anymore, and there's no visualization that would really help here as far as I can tell.


## The acoustic model

The whole thing got inspired by [Matcha-TTS.](https://arxiv.org/abs/2309.03199) The pipeline is like this:
* First, the encoder reads the phone sequence and, for each phone, produces a guess of what its slice of the spectrogram looks like (there's one mel vector per phone,) and how long the phone should last.  Take `cat` as an example, there's three phones, `/k æ t/`.  For each phone, the encoder will give us a 100 dimensional vector.
* Second, there's an alignment search that figures out at training which frames of the final spectrogram belong to which phone.  So dynamically, we need to answer "`/k/` will last 8 frames, `/æ/` 15 frames, and `/t/` only 6 frames."  It's entirely based on [Glow-TTS.](https://arxiv.org/abs/2005.11129)
* Then, a flow-matching decoder learns to tweak the guess and build the spectrogram.  It works like a diffusion model: at training you teach the model to go from noise to the final picture, and at inference time you go from noise to whatever directions the network tells us to go to.

It works pretty well given that we produce speech from the get go: the encoder's guess is already a spectrogram.  We then just need to run it through the vocoder and we can hear it talk: and that's way before the decoder is even good (it's just not great synthesis.)  As an example, here's a sentence the model never saw in training:
* here's the encoder's guess sent to the vocoder: <audio controls preload="none" src="/images/vovo/probe-prior.m4a"></audio> ![This is the encoder's mel spectrogram](/images/vovo/probe-prior-mel.png)  It's more of staircase sort of signal, with sharp edges.
* and here's what the decoder gives: <audio controls preload="none" src="/images/vovo/probe-freerun.m4a"></audio> ![Now the decoder's mel spectrogram](/images/vovo/probe-freerun-mel.png)  You can see more details added by the decoder, the spectrogram has more pitch curves and some texture.


## The vocoder

As mentioned earlier we've reused Vocos for this.  Vocos takes the decoder's spectrogram and generates the waveform.  There was some extra work to map the weight files to my custom design but I honestly left that part to the coding agents... it was kind of a self-inflicted curveball to go without python but it worked out just fine.  I'll probably publish how I had to fine-tune Vocos to Vovo's own output, it did make a difference.

## The result

As you can hear it already above, I got a voice up and running in almost no time.  The word error rate (aka WER) was already low (5 words out of 241 were off, so about 2%.)  It doesn't sound like a state of the art model, but at the end of the day it's still intelligible, so a win in my book.

Try it out, `pip install vovo-mlx` then `vovo-mlx say "..."` (or use the Swift package directly with the weights from Huggingface.)