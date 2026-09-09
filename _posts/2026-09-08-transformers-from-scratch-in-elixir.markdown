---
layout: post
title: "Transformers from Scratch, in Elixir"
date:   2026-09-08 20:00:00 -0400
tags:   elixir,machinelearning,ai
---

Fill in the blank: **flees**, or **flee**?

![the llama who chases the dogs: flees, or flee](/assets/tiny-llm-the-vote.jpg)

Everybody gets it. The rule is easy to state: the verb agrees with its
subject. The hard part is knowing which noun is the subject when a plural one
is sitting right next to the blank. That was the whole of a talk I gave this
week: one sentence, one forward pass through a transformer small enough to
read, and the question of whether it learned that rule.

The model is [tiny_llm](https://github.com/jasondew/tiny_llm). It is pure
Elixir with an empty dependency list: no Nx, no GPU, no tokenizer. Thirty-two
words, one attention head, one block, 15,104 floats. It trains in about 77
seconds on a laptop, and it did so live on the first slide while the room was
arriving. The slides are a Phoenix LiveView app,
[tiny_llm_talk](https://github.com/jasondew/tiny_llm_talk), and every number
on every slide is read from the trained model at render time.

## Where will the blank look?

The blank is predicted from the last position given, *dogs*, so I asked the
room: which word does the *dogs* row attend to most? Most people said
*llama*, the subject. The model said **who**, at 64.8%.

![where will the blank look](/assets/tiny-llm-where-the-blank-looks.jpg)

That looked wrong to me too, until I looked at the *who* row: 68.5% of its
attention is on *llama*. The blank reaches the subject in two hops, through
the pronoun that stands for it. Nobody designed that. It fell out of
training, and it is a small, checkable example of why these models feel
strange from the outside.

## It learned it

Training is five sentences: take a prefix from the corpus where we know the
next word, run the model, measure how surprised it was by the real word,
nudge every number in the direction that makes the surprise smaller, repeat
a few hundred times. The corpus comes from a grammar I wrote, so "did it
learn agreement across a relative clause" is a measurement, not a vibe.

![training, live](/assets/tiny-llm-training-live.jpg)

The dashed line at 1.904 is the best any model can do by looking only at the
previous word. The transformer lands at 1.577. That gap is the model using
information the previous word does not carry, which for this sentence means
looking past *dogs* to *llama*. On the sentence itself it puts *flees* at
18.7% and *flee* at 14.7%. Not a landslide, but the right way round.

## Watching it write

The deck ends with the model writing a paragraph one word at a time, with
its forward pass drawn beside each word.

![the model writing, with its forward pass beside it](/assets/tiny-llm-writer.gif)

What is not in it: a tokenizer, a GPU, multi-head attention, depth, KV
caching. Everything else that is in a frontier model is in this one. The
difference is thirteen orders of magnitude and a tokenizer.

Both repos are public:
[github.com/jasondew/tiny_llm](https://github.com/jasondew/tiny_llm) and
[github.com/jasondew/tiny_llm_talk](https://github.com/jasondew/tiny_llm_talk).
Run `mix phx.server` and press start.
