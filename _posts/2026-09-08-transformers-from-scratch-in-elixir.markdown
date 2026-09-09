---
layout: post
title: "Transformers from Scratch, in Elixir"
date:   2026-09-08 20:00:00 -0400
tags:   elixir,machinelearning,ai
---

Fill in the blank: **flees**, or **flee**?

![the llama who chases the dogs: flees, or flee](/assets/tiny-llm-the-vote.jpg)

The answer is **flees** but why? The verb needs to agree with the subject, but
think about what the model needs to look in to understand that relationship. It
doesn't look at the previous word but several words back to "llama." The
attention mechanism is how it learns that.

I gave a talk to our AI guild today called *Transformers from Scratch, in
Elixir* explaining the attention mechanism and also the basic transformer
architecture.

A large part of why I decided to give the talk was to more deeply understand
attention and the transformer achitecture myself. So, I decided to implement a
very small model in Elixir, completely from scratch -- no libraries allowed. I
cut the vocabulary down to just 32 words, used a single block and head of
attention. It comes in at just over 15k parameters and trains in ~70 seconds on
my laptop and gets the noun verb agreement correct. The source code is in
Github at [tiny_llm](https://github.com/jasondew/tiny_llm).

For the talk, I created a Phoenix LiveView app that actually runs the model
live.

## A language model is one function

At the highest level, an LLM is a function that takes a sequence of words and
returns a probability distribution.

![the words so far, a language model, 32 probabilities](/assets/tiny-llm-one-function.jpg)

A transformer is one block, repeated. Embeddings plus positions on the way
in, then normalization, attention, normalization, and a small feed forward
network, then a projection back to one score per word. Frontier models stack
that block dozens of times with many heads. This one stacks it once, with one
head, which is what makes it readable.

![the transformer in this talk](/assets/tiny-llm-the-transformer.jpg)

## Attention is a fuzzy lookup

Attention is the part people have heard of, and it is less than it sounds.
Every position asks a question (a query), every position advertises what it
has (a key), and every position offers something to hand over (a value). Dot
the query against every key, softmax the scores into shares, and blend the
values by those shares. That is the whole formula.

I did it by hand first, on a five-entry map with two-number vectors, asking
"is it plural?" The query *geese* is not a key in the map, and it still gets
the right answer: the blend lands on 1.00, plural, because *geese* is close
to *foxes* and *mice* in the two numbers that matter.

![a small example by hand](/assets/tiny-llm-fuzzy-map.jpg)

Then the real thing: one head of attention is sixteen lines of Elixir, shown
with the actual 7 × 7 weight matrix for the sentence beside it.

![one head of attention, sixteen lines](/assets/tiny-llm-attention-code.jpg)

## Where will the blank look?

I asked the room to bet. The blank is predicted from the last position given,
*dogs*, so the question is: which word does the *dogs* row attend to most?
Most people say *llama*, since that is the subject. The model says **who**,
at 64.8%.

![where will the blank look](/assets/tiny-llm-where-the-blank-looks.jpg)

That looked wrong to me too, until I looked at the *who* row: it puts 68.5%
of its attention on *llama*. The blank reaches the subject in two hops,
through the relative pronoun that stands for it. Nobody designed that. It fell
out of training, and it is a small, checkable example of the thing that makes
these models feel strange from the outside.

## The rest of the block

The other pieces are less famous and each is a few lines. RMSNorm divides a
row by its typical size so the next stage sees rows at one scale. The feed
forward network is two matrix multiplies with a ReLU between them, and it
holds more than half the parameters. The residual connections add each
stage's output back to its input rather than replacing it, which is what lets
deep stacks train at all. Here is the whole block:

![Block.forward, nine lines](/assets/tiny-llm-block-forward.jpg)

And here is the entire forward pass, seven lines from embeddings to logits:

![this was all of it](/assets/tiny-llm-all-of-it.jpg)

Thirty-two scores go through a softmax, with a temperature knob, and become
the 32 probabilities from the first slide. For our sentence: *flees* 18.7%,
*are* 17.0%, *flee* 14.7%. Not a landslide, but *flees* beats *flee*, and it
beats it because of *who*.

![thirty-two floats become thirty-two probabilities](/assets/tiny-llm-back-to-words.jpg)

## It learned it

Training is five sentences: take a prefix from the corpus where we know the
next word, run the model, measure how surprised it was by the real word,
nudge every number in the direction that makes the surprise smaller, repeat
a few hundred times. The corpus is 2,000 sentences written by a grammar I
wrote, so "did it learn subject-verb agreement across a relative clause" is a
measurement, not a vibe.

The run happens live on the first slide while the room is arriving, in the
same BEAM as the deck, with the checkpoint's seed. Because the model is pure
Elixir over a seeded random number generator, it is the same run every time,
and the slide says whether it matched.

![training, live](/assets/tiny-llm-training-live.jpg)

The dashed line at 1.904 is the best any model can do by looking only at the
previous word. The transformer lands at 1.577. That gap is the model using
information the previous word does not carry, which for this sentence means
looking past *dogs* to *llama*.

## Watching it write

The deck ends with the model writing a paragraph, one word at a time, with
its forward pass drawn beside each word. Integers, rows, query against keys,
the attention shares, the blend, the rest of the block, one draw.

![the model writing, with its forward pass beside it](/assets/tiny-llm-writer.gif)

## What is not here

A tokenizer, a GPU, multi-head attention, depth, KV caching. Everything else
that is in a frontier model is in this one: embeddings, learned positions,
scaled dot-product attention, a causal mask, residuals, RMSNorm, a feed
forward network, temperature sampling. The difference is thirteen orders of
magnitude and a tokenizer.

Both repos are public. The model is
[github.com/jasondew/tiny_llm](https://github.com/jasondew/tiny_llm) and the
deck is
[github.com/jasondew/tiny_llm_talk](https://github.com/jasondew/tiny_llm_talk).
Run `mix phx.server` and press start.
