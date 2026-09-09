---
layout: post
title: "Transformers from Scratch, in Elixir"
date:   2026-09-08 20:30:00 -0500
tags:   elixir,machinelearning
---

Fill in the blank: **flees**, or **flee**?

![the llama who chases the dogs: flees, or flee](/assets/tiny-llm-the-vote.jpg)

Everybody gets it. The llama flees. The rule is easy to state: the verb
agrees with its subject. The hard part is knowing which noun is the subject
when a plural one is sitting right next to the blank. That was the whole
talk I gave this week, *Transformers from Scratch, in Elixir*: one sentence,
one forward pass through a transformer small enough to read, and the
question of whether it learned that rule.

The model is [tiny_llm](https://github.com/jasondew/tiny_llm). It is pure
Elixir with an empty dependency list: no Nx, no GPU, no tokenizer. Thirty-two
words, one attention head, one block, 15,104 floats in total. It trains in
about 77 seconds on a laptop. The slides are a Phoenix LiveView app,
[tiny_llm_talk](https://github.com/jasondew/tiny_llm_talk), and every number
on every slide is read from the trained model at render time, so nothing on
screen is a drawing of what the model does. It is what the model does.

## A language model is one function

The frame for the whole talk: the words so far go in, and 32 probabilities
come out, one per word. Everything else is inside the box.

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
