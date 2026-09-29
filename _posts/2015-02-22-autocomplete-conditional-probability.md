---
layout: post
title: "Autocomplete: From Word Counts to Conditional Probability"
date: 2015-02-22
author: Rodrigo Palacios
description: "How a simple word-frequency model becomes a next-word predictor by conditioning on the previous word."
excerpt: "Counting words gives you a start. Counting which words follow other words gives you an autocomplete model. Here's the intuition and the code behind mine."
tags: [Python, autocomplete, Markov chains, probability]
categories: [Python, Algorithms]
---

*Adapted from the [conditional probability explanation in my autocomplete README](https://github.com/rodricios/autocomplete#tldr), first published February 22, 2015.*

I built [autocomplete](https://github.com/rodricios/autocomplete) because it took me far too long to find a palpable theory-to-application example of a simple predictive model. The books had the equations. I wanted to see what those equations looked like when the problem was: *what word is somebody trying to type next?*

The answer starts with counting. It just doesn't end there.

## First, count words

Take a large body of text, split it into words, and normalize them so `The` and `the` count as the same word. A `Counter` gives us a frequency distribution:

```python
from collections import Counter

words = ["the", "third", "thing", "the", "third", "time"]
word_counts = Counter(words)
# Counter({'the': 2, 'third': 2, 'thing': 1, 'time': 1})
```

If you type `th`, I can filter that distribution to words beginning with `th` and rank them by count. That is already a primitive autocomplete. It may suggest `the`, `they`, and `there` before `third`, because those words are common across the *whole* corpus.

But imagine you typed `the th` and you meant `the third`. The first model knows about the fragment `th`; it knows nothing about the word before it. It models **P(word)**, the probability of a word without context.

That's a useful baseline. It's also the point where we need a second count.

## Count what follows what

Instead of counting only individual words, walk through the corpus in overlapping pairs:

```text
[the, third, thing, the] → (the, third), (third, thing), (thing, the)
```

For each first word, count the words that came after it. Now `the` points to its own, smaller frequency distribution. That is how I interpreted the word *given* in **P(next word | previous word)**: the previous word is the key that tells us which distribution to inspect.

For example, if `the` is followed by `third` 239 times and by `thing` 303 times, those counts compete inside the distribution for `the`. Counts from unrelated contexts don't get a vote. When the previous word is fixed, dividing every count by the same total would turn them into probabilities without changing their order.

The familiar identity is:

```text
P(previous word, next word)
    = P(next word | previous word) × P(previous word)
```

And the conditional probability can be estimated directly from the corpus:

```text
P(next word | previous word)
    = count(previous word, next word) / count(previous word)
```

This is the Markov idea in miniature. Treat each word as a state; count transitions from one state to the next. To predict the next word, use the current word as context. My model looks back **one word**. It doesn't remember the whole sentence, which is both what makes it simple and what limits it.

## In Python

Here is the core of the model, stripped down to the part that explains the theory:

```python
import re
from collections import Counter, defaultdict


def train(corpus):
    words = re.findall(r"[a-z]+", corpus.lower())
    word_counts = Counter(words)
    next_word_counts = defaultdict(Counter)

    for previous, following in zip(words, words[1:]):
        next_word_counts[previous][following] += 1

    return word_counts, next_word_counts


def suggest(previous, prefix, next_word_counts, limit=5):
    candidates = (
        (word, count)
        for word, count in next_word_counts[previous.lower()].items()
        if word.startswith(prefix.lower())
    )
    return sorted(candidates, key=lambda item: (-item[1], item[0]))[:limit]


corpus = "the third thing was the third example. the thing worked."
word_counts, next_word_counts = train(corpus)
print(suggest("the", "th", next_word_counts))
# [('third', 2), ('thing', 1)]
```

The [repository's implementation](https://github.com/rodricios/autocomplete/blob/master/autocomplete/models.py) stores those two distributions as `WORDS_MODEL` and `WORD_TUPLES_MODEL`. Its prediction function also filters candidates by the letters typed so far, with a small nearby-key heuristic for a mistyped final letter. The code above leaves that spelling layer out so the conditional model is easier to see.

## Why the prefix matters

Knowing the previous word narrows the distribution; knowing the first letters of the next word narrows it again. So for `the th`, I look at words observed after `the`, keep only those starting with `th`, and rank them by their counts in that context.

You can make the context longer, of course. Count triples instead of pairs and ask what followed the previous *two* words. But now you need a bigger corpus: longer sequences occur less often, and your model will have more cases where it has seen no continuation at all. One previous word is a modest place to start.

That's the thing I wanted from the project: a small bridge from an equation in a book to a program you can read. It won't guess every word you meant. It does make the reasoning visible, and that's enough to start improving it.
