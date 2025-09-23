---
layout: post
title: Vibecoding Triton
date: 2025-09-22
---

I've seen some discussions on Twitter about vibecoding Kernels recently, and felt the need to weigh in, specifically about Triton kernels.
I'm pretty bearish on vibecoding Triton kernels, and kind of doubt anyone who claims that they've produced SoTA performance with just Claude or some good prompts. 
Why is that? 

# Benchmarking is Hard
I don't believe these people are using real workloads, or understand their performance enough to really be driving real results here. 
I'd love to write more about this particular problem, but every good performance engineer I've talked to has shared two beliefs:

1. The best way to tell if something will improve the performance on your problem is to benchmark it.
2. Benchmarking is hard.

I'm pretty sure a lot of these good results aren't comparing to AOTInductor, or Max Autotune, or are specializing on aligned problems or ignoring different strides or all kinds of complicated corner cases that come up when you're designing for real models that the LLMs aren't good at, because they aren't in the training corpus. 
This is because the training corpus is mostly examples, which brings me to the second reason I'm doubtful.

# My experience Vibecoding some kernels
In my professional job, I'm working on improving model performance, and I came across a portion of a module we were designing that seemed like a good candidate for a custom fused kernel. 
I tried to use our AI tooling to write a custom kernel.
The result? A wasted day, a bunch of wasted agent tokens and a kernel that while mathematically correct, was worse performing than `torch.compile` and even torch eager mode.

## What worked poorly at first
I spent over a day trying to debug kernels that didn't compile, didn't start up or had CUDA IMAs. 
Prompting the coding agents with the equivalent torch code got half-working guesses that were clearly based on triton examples. 
However, the agent would often hallucinate `tl` APIs based on `torch` or `np` APIs, and clearly didn't understand the specifics of the DSL.
In addition, the agent that I used was super overactive, and would often try to fix an entire file in one edit, which would cause it to go down loops and fail.

## What worked well
Prompting the kernel with a really specific design based on a specific triton example got it to at least write compilable code. 
Having specific references to the triton tutorials (in my case saying I wanted to write a persistent kernel) seemed to let it focus on writing the code and not hallucinating a solution to the problem with APIs that don't exist.
With a lot of prompting and a clear equivalence suite, I eventually massaged it into numerically correct code. 
However, I really doubt that there's any hope for the agent to have a novel insight on the memory accesses, block sizes or pipelines and suggest a new approach. 
If I have some free time I'll probably go look at the kernel under NCU and see what the bottlenecks are now and work from there, but I don't think the agents will be helpful going forward.

To me this begs the question, why Triton? 
If I have to tell the agent the specific approach I want it to write anyway, what benefit are we getting from Triton being a high level language? 
Shouldn't the LLM just produce CUDA C++ code (or cutlass code or whatever).
If you're (Tri Dao)[https://x.com/scaling01/status/1970146206203416666] and you've forgotten more about GPU architecture then I've ever learned, maybe it helps you skip some boilerplate, but for most ML engineers I would not suggest vibecoding Triton, and I'd be really skeptical of the people claiming sick improvements from AI written kernels.