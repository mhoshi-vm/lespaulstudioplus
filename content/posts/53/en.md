---
title: "Notes on Running Qwen3.8-Flash-Next on an A100 (40GB) + Strata"
date: "2026-10-11T19:00:00+09:00"
tags: ["PrivateAI", "LLM", "GPU", "AI", "A100", "Qwen", "Strata"]
thumbnail: img_2.png
---
By using an A100 + Strata, I'm now getting a blazing 140-150 Tokens/s out of a 125B model.
<!--more-->

## Trying Out Strata

[Strata](https://github.com/Niko1221/Strata/blob/main/README.md) has been getting a lot of attention lately.
It lets you run Qwen3.8-Flash-Next, a monster LLM with 125B parameters, even on consumer GPUs.

In the [previous post](../52), I worked hard to get Qwen3.8-27B running on an A100, so could Strata really run something this big?
I had to try it.

## 140 Tokens/s !?

Jumping straight to the result: I ran it on the A100 with Strata and tried the same code generation as in the previous post, and as you can see below, it is blazing fast...

![A100+Strata.gif](A100%2BStrata.gif)

Looking at the speed, it's 140 Tokens/s, which is roughly 3-4 times the performance I got with Qwen3.8-27B last time.

![img.png](img.png)

By the way, Image-to-Text works too. Below, it is transcribing a sample image via OCR, and this speed... it's almost scary.

![OCR.gif](OCR.gif)

Here are some assorted impressions.

- Accuracy went up and speed went up too. What is going on!?
- For a while I considered vLLM the strongest way to serve a model, but that belief has collapsed
- The A100's memory bus transfer speed helps a great deal, and the result exceeds the benchmark numbers Strata publishes
- It's even easier to install than vLLM
- Following the installation steps, the KV Cache is set to 128k for now. Increasing the KV Cache probably requires multiple GPUs
- I had started to feel the limits of Qwen3.8-27B, so it feels like I've gained a new weapon

On top of that, my understanding of memory in particular changed a lot, so I'll add some notes on that as well.

## Infrastructure

I assigned a virtual machine like the following.

- CPU 8 Cores / Xeon E5-2697 v4
- Memory 64GB
- Disk 150GB / NVMe
- A100 40GB assigned via passthrough

As you can see, the CPU model is quite old, so I expected it to be the bottleneck, but that doesn't seem to be the case.


## RAM vs vRAM

Qwen3.8-Flash-Next is close to 70GB in model size alone.
The A100 40GB, on the other hand, has only 40GB of vRAM, just as the name says.

With only my vLLM-based knowledge from last time, I assumed that "the model is loaded into vRAM, so a model larger than 40GB cannot be hosted."
On top of that, the KV Cache also lives in vRAM, so I was convinced that running any reasonable model other than Qwen3.8-27B on a single A100 (40GB) was impossible.
In fact, even with Qwen3.8-27B, I misconfigured the KV Cache a few times and overflowed vRAM into RAM, and whenever that happened
I saw extreme performance degradation. Those experiences left me with the feeling that "overflowing vRAM means you lose."

Strata takes a different approach: it uses not only vRAM but also RAM efficiently.
It appears to take advantage of the characteristics of Mixture of Experts (MoE) models and place only what is truly needed in vRAM.
As a result, part of the model and the KV Cache are offloaded to RAM, but you can barely notice it, and it powers through at tremendous speed.

A model with over 100B parameters, on a single card with around 40GB of vRAM, at this speed... it feels like we've entered an incredible era.


## Summary

With Strata, the A100 has entered a new era...
