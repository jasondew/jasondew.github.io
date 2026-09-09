---
layout: post
title: "Transformers from Scratch, in Elixir"
date:   2026-09-08 20:00:00 -0400
tags:   elixir,machinelearning,ai
---

I gave a talk to the AI guild at work today called *Transformers from Scratch,
in Elixir* explaining attention and the transformer architecture.

A large part of why I decided to give the talk was to more deeply understand
attention and the transformer architecture myself. So, I decided to implement a
very small model in Elixir, completely from scratch -- no libraries allowed. I
cut the vocabulary down to just 32 words, used a single block and head of
attention. It comes in at just over 15k parameters and trains in about 75
seconds on my laptop and gets the noun verb agreement correct.

The part that surprised me: for *the llama who chases the dogs ____*, the
position predicting the blank puts 65% of its attention on *who*, not on
*llama*. Then *who* puts 68% of its attention on *llama*. The model reaches the
subject in two hops instead of directly. LLMs are unintuitive!

![where will the blank look](/assets/tiny-llm-where-the-blank-looks.jpg)

For the talk, I created a Phoenix LiveView app that actually runs the model
live. Here's it running, showing its work:

![the model writing, with its forward pass beside it](/assets/tiny-llm-writer.gif)

The talk covered embeddings, position embeddings, attention, feed forward networks, and the transformer architecture. Both repos are public if you'd like to check them out!

- [github.com/jasondew/tiny_llm](https://github.com/jasondew/tiny_llm)
- [github.com/jasondew/tiny_llm_talk](https://github.com/jasondew/tiny_llm_talk)
