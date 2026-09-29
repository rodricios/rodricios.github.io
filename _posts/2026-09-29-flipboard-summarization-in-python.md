---
layout: post
title: "Flipboard's Summarization Algorithm, Sort Of, in Usable Python"
date: 2026-09-29
author: Rodrigo Palacios
description: "A small, runnable Python 3 implementation of graph-based extractive summarization, revisiting my Flipboard-inspired gist."
excerpt: "Back in 2015 I translated Flipboard's description of extractive summarization into a Python gist. Here's the idea again, this time as code you can run today."
tags: [Python, summarization, algorithms]
categories: [Python, Algorithms]
---

Back in 2015, I read [Flipboard's explanation of automatic summarization](https://engineering.flipboard.com/2014/10/summarization) and thought: this is unusually close to executable code. So I [wrote a gist](https://gist.github.com/rodricios/fee45381356c8fb36004) that followed the description almost step by step.

The gist was fun, but its dependencies and Python syntax belong to a different era. If you want to try the idea today, you shouldn't have to resurrect an old environment first.

So let's do it again.

**TL;DR:** Make every sentence a node in a graph. Connect sentences that share words. Rank the nodes. Take the highest-ranked sentences, then put them back in the order they appeared in the article.

## Why a graph?

Suppose three sentences talk about web crawlers and one sentence talks about my basil plant. The first three will share words with each other. The basil sentence probably won't. If I'm trying to summarize the *whole* passage, the connected sentences are a reasonable place to look.

Flipboard described this as **extractive summarization**: the summary is made of sentences already in the text. There is no model generating a new sentence. That doesn't guarantee a *good* summary, but it does mean the algorithm won't invent a sentence that wasn't there.

The recipe has four parts:

1. Split the text into sentences and turn each sentence into a set of words.
2. Compare every pair of sentences. I use Jaccard similarity: the number of shared words divided by the number of distinct words across the pair.
3. Use those similarities as weighted graph edges. Normalize each node's outgoing weights so they sum to one.
4. Run PageRank, select the top `n` sentences, and restore their original order.

My old gist skipped the normalization *explicitly* because NetworkX handled the weighted transitions inside `pagerank`. Here the normalization is visible in the code.

## The code

Save this as `summarize.py` and run it with Python 3. It uses only the standard library.

```python
import re
from itertools import combinations


SENTENCE_END = re.compile(r"(?<=[.!?])\s+")
WORD = re.compile(r"[a-z0-9]+(?:'[a-z0-9]+)?")


def summarize(text: str, n: int = 2) -> str:
    sentences = [
        sentence.strip()
        for sentence in SENTENCE_END.split(text.strip())
        if sentence.strip()
    ]
    if not sentences or n <= 0:
        return ""

    count = len(sentences)
    word_sets = [set(WORD.findall(s.casefold())) for s in sentences]
    edges = [{} for _ in sentences]

    for i, j in combinations(range(count), 2):
        shared = word_sets[i] & word_sets[j]
        all_words = word_sets[i] | word_sets[j]
        similarity = len(shared) / len(all_words) if all_words else 0
        if similarity:
            edges[i][j] = similarity
            edges[j][i] = similarity

    damping = 0.85
    scores = [1 / count] * count
    for _ in range(100):
        updated = [(1 - damping) / count] * count
        for i, neighbors in enumerate(edges):
            total_weight = sum(neighbors.values())
            if total_weight:
                for j, weight in neighbors.items():
                    updated[j] += damping * scores[i] * weight / total_weight
            else:
                # A sentence with no neighbors distributes its vote evenly.
                for j in range(count):
                    updated[j] += damping * scores[i] / count

        if sum(abs(a - b) for a, b in zip(scores, updated)) < 1e-10:
            scores = updated
            break
        scores = updated

    ranked = sorted(range(count), key=lambda i: (-scores[i], i))
    chosen = sorted(ranked[:n])
    return " ".join(sentences[i] for i in chosen)


if __name__ == "__main__":
    article = (
        "Web crawlers fetch pages and follow links across a site. "
        "Extractors turn those pages into structured data. "
        "A crawler and an extractor work together to collect web data. "
        "Yesterday I watered the basil on my windowsill."
    )
    print(summarize(article, n=2))
```

There's a small detail in the pairwise loop that's worth calling out. A commenter on the gist suggested `itertools.combinations`, while another suggested `permutations`. For this graph, `combinations` is enough **if** we add the edge in both directions. The similarity between sentence A and sentence B is the same either way; we don't need to calculate it twice.

## What this gets wrong

Quite a bit, potentially! Sentence splitting on punctuation will stumble over abbreviations. The tokenizer won't recognize that *crawler* and *crawlers* are related. Common words can create weak links that aren't meaningful. And an extractive summary can repeat itself or miss the sentence you actually cared about.

The original gist used `pattern` for tokenization and lemmatization, and discarded very short sentences. I left those choices out to keep this version easy to run and inspect. If I were putting it into a product, I'd start by improving sentence boundaries and word normalization, then evaluate it on real articles before claiming the summaries are any good.

Still, I like this algorithm. It's small enough to understand, it has a clear reason for every step, and it gives you a baseline to beat. Sometimes that's more useful than starting with a black box.

This is my **[LexRank](https://www.cs.cmu.edu/afs/cs/project/jair/pub/volume22/erkan04a-html/erkan04a.html)-inspired interpretation** of [Flipboard's writeup](https://engineering.flipboard.com/2014/10/summarization), not Flipboard's production implementation. The [original gist](https://gist.github.com/rodricios/fee45381356c8fb36004) is still there if you want to see where I started.
