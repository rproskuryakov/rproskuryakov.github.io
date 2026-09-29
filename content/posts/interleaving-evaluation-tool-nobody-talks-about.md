+++
title = "Interleaving, a Retrieval Online Evaluation Method Nobody Talks About. Part 1"
date = "2026-09-29"
draft = false
description = "Why A/B-tests struggle to evaluate ranking models, and how interleaving, starting with balanced interleaving, detects the better ranker with far less traffic."
[taxonomies]
tags=["search", "A/B-testing", "experimenting", "interleaving"]
[extra]
comment = true
mermaid = true
+++


## Introduction

A/B-testing has become a widely adopted tool for online evaluation of machine learning models across the industry in the past 15 years.
Despite its advantages, there are still lots of issues to tackle.
The main culprit of any A/B-test is variance.
Multiple efforts have been made throughout recent years to mitigate the problem.
For instance, methods such as [CUPED](https://exp-platform.com/Documents/2013-02-CUPED-ImprovingSensitivityOfControlledExperiments.pdf) were developed
to reduce variance by using pre-experiment data as a covariate.

Yet, when you work on search or recommendations, even a well-tuned A/B-test often needs weeks of traffic
to detect a change in ranking quality.
If your team produces several ranking hypotheses a week, the experimentation platform quickly becomes the bottleneck.

There is another method to reduce variance in such cases that almost nobody talks about: interleaving.
Originally, it was developed to test web search engines, but it can be applied to online tests of any ranking model.
Companies such as Netflix, Airbnb, DoorDash and Amazon report that it detects the winning ranker
with one or two orders of magnitude less traffic than a classic A/B-test.

## Who is it for?

This post is for ML engineers, search engineers and data scientists who run online experiments
on search ranking or recommender systems and want to iterate faster.
I assume you are familiar with the basics of A/B-testing and hypothesis testing.
No prior knowledge of interleaving is required.

## Overview

This is the first post of a series on interleaving.
In this part, I will walk you through:
* why A/B-tests struggle with ranking models;
* the idea behind interleaving and the first method, balanced interleaving;
* the general framework every interleaving method fits into: interleaving policy and credit attribution;
* the bias that makes balanced interleaving unreliable.

In the second part, coming soon, we will look at Team-Draft interleaving,
the limitations of interleaving and how the industry uses it.
So, let's get started!

## Why A/B-tests struggle with ranking

Let's look at the sources of variance in testing a search engine.

### Users are different

In a classic A/B-test, the population is split into two groups: one is served by ranker A, the other by ranker B.
The problem is that users contribute very differently to the metric.
A heavy user who makes fifty searches a day and a casual user who searches once a week land in the groups randomly,
and the difference in their behaviour is often much larger than the difference between the rankers.

Think of a cola taste test.
You could give brand A to one group of people and brand B to another group and compare how much they liked it.
Or you could give both glasses to the same person and ask which one tastes better.
The second design removes the difference between people from the comparison entirely,
and you need far fewer participants to get a confident answer.

### Queries are different

The same applies to queries.
Some queries are navigational and both rankers return the same obvious result at the first position.
Others are ambiguous and no ranker performs well on them.
Only a fraction of queries are *competitive*, meaning that the rankers produce meaningfully different results.
In an A/B-test, the non-competitive queries still contribute noise to the metric, but no signal.

### Randomization unit

Finally, one should choose a randomization unit: user, session or query.
Randomizing by user is the most common choice as it keeps the experience consistent,
but it maximizes the between-user variance described above.
Randomizing by query reduces variance, but breaks the user experience and makes long-term metrics impossible to measure.

Interleaving takes the "same person tastes both glasses" approach.
Every user sees a single result list that combines the outputs of both rankers,
and the user's own clicks tell us which ranker they prefer.

{% mermaid() %}
flowchart LR
    subgraph AB["A/B-test"]
        direction TB
        U1["Users"] --> S{"Random split"}
        S -->|50%| RA["Ranker A"]
        S -->|50%| RB["Ranker B"]
        RA --> MA["Metric of group A"]
        RB --> MB["Metric of group B"]
    end
    subgraph IL["Interleaving"]
        direction TB
        U2["Users"] --> Q["Query"]
        Q --> A2["Ranker A"]
        Q --> B2["Ranker B"]
        A2 --> M["Interleaved list"]
        B2 --> M
        M --> C["Clicks credited to A or B"]
    end
{% end %}

## The idea of interleaving

The idea behind interleaving was introduced by Thorsten Joachims in 2002 in
[Evaluating Retrieval Performance using Clickthrough Data](https://www.cs.cornell.edu/~tj/publications/joachims_02b.pdf).

Absolute click metrics are hard to interpret: users click on what they see first,
and the click-through rate depends heavily on the query.
Joachims proposed to ask a *relative* question instead: given a combined list of results from both rankers,
does the user click more on results from A or from B?

Let $C_a$ and $C_b$ be the number of clicks on results coming from rankers A and B for a query,
and $C$ the total number of clicks.
Joachims showed that under reasonable assumptions the relevance of the rankers can be compared by estimating
$E\left(\frac{C_a - C_b}{C}\right)$.
If this value is significantly bigger than zero, ranker A produces more relevant results than ranker B.

The method relies on two assumptions:
* users click on relevant results more often than on non-relevant ones;
* users do not click more frequently on results of one ranker independently of their relevance.
In other words, the combined list must be *blind*: the user cannot tell which ranker produced which result.

The experiments in the paper support both assumptions,
and the outcome of interleaving agrees with manual relevance judgements.

There is also a nice side effect.
In an A/B-test, half of the users are fully exposed to the treatment, even if the new model turns out to be terrible.
With interleaving, a bad ranker contributes only part of each result list, so the damage to user experience is limited.

## Balanced interleaving

The first interleaving method is called balanced interleaving.

### Interleaving policy

Formally, let $A = (a_1, a_2, ..., a_n)$ and $B = (b_1, b_2, ..., b_n)$ be the outputs of two ranking models.
Let $I = (i_1, i_2, ..., i_n)$ be the combined ranking.

Balanced interleaving flips a coin once per query to decide which ranker goes first.
Then it takes results from A and B alternately, skipping the ones already present in $I$.
The resulting list has a nice property: for any cut-off $k$, the top-$k$ of $I$ contains the top-$k_a$ results of A
and the top-$k_b$ results of B, where $k_a$ and $k_b$ differ by at most one.

```python
import random


def balanced_interleave(a: list, b: list, length: int) -> list:
    a_first = random.random() < 0.5
    result = []
    ka, kb = 0, 0
    while (ka < len(a) or kb < len(b)) and len(result) < length:
        a_turn = ka < kb or (ka == kb and a_first)
        if kb >= len(b) or (ka < len(a) and a_turn):
            if a[ka] not in result:
                result.append(a[ka])
            ka += 1
        else:
            if b[kb] not in result:
                result.append(b[kb])
            kb += 1
    return result
```

For example, let A = (a, b, c) and B = (b, d, a).
Depending on the coin, the user sees one of two lists:

{% mermaid() %}
flowchart TD
    Q["Query"] --> Coin{"Coin flip"}
    Coin -->|"A first"| L1["1. a from A<br/>2. b from B<br/>3. d from B<br/>4. c from A"]
    Coin -->|"B first"| L2["1. b from B<br/>2. a from A<br/>3. d from B<br/>4. c from A"]
{% end %}

Note that b, which is ranked highly by both models, appears only once, and the ranker that would add it second simply skips it.

### Credit attribution

Let $c_1, c_2, ..., c_m$ be the ranks of the clicked results in $I$ and $c_{max}$ the largest of them, i.e. the lowest clicked position.
Joachims proposes to look only at the part of both rankings the user has actually examined.
Let

$$k = \min \lbrace j : i_{c_{max}} = a_j \lor i_{c_{max}} = b_j \rbrace$$

Then the clicks attributed to A are the clicked results that appear among the top-$k$ results of A, $h_a$,
and the same for B, $h_b$.
If $h_a > h_b$, ranker A wins the query; if $h_a < h_b$, B wins; otherwise it is a tie.

### Statistics

Having collected the outcomes, we need to decide whether one ranker is significantly better.

Joachims suggests a two-tailed paired t-test on per-query samples $h_a/c$ and $h_b/c$.
In practice, collecting samples that hold the assumptions of the t-test can be challenging.
In such cases, the author proposes an alternative approach, the binomial sign test,
which only looks at the number of queries won by each ranker.

Later works aggregate the outcomes into a single preference score.
Let $W_A$ and $W_B$ be the number of queries won by A and B correspondingly, and $T_{AB}$ the number of ties:

$$ \Delta_{AB} = \frac{W_A + \frac{1}{2}T_{AB}}{W_A + W_B + T_{AB}} - 0.5$$

A positive value of $\Delta_{AB}$ indicates that A is better than B, a negative value indicates that B is better.
Queries without clicks carry no information and are excluded.

To get a confidence interval of $\Delta_{AB}$, the bootstrap is commonly used.
Subsamples are formed by drawing queries, sessions or users with replacement, $\Delta_{AB}$ is computed for each subsample,
and the percentile method gives the interval.
Resampling by user rather than by query accounts for the fact that queries of the same user are correlated.

## General framework

Balanced interleaving highlights two design decisions every interleaving method has to make:
* **interleaving policy**: how the combined list is constructed from the compared rankings;
* **credit attribution**: how user actions are attributed back to the compared rankers.

{% mermaid() %}
sequenceDiagram
    actor User
    participant I as Interleaver
    participant A as Ranker A
    participant B as Ranker B
    participant L as Experiment log
    User->>I: query
    I->>A: query
    I->>B: query
    A-->>I: ranking A
    B-->>I: ranking B
    Note over I: interleaving policy
    I-->>User: combined list
    I->>L: list with the origin of every result
    User->>L: clicks
    Note over L: credit attribution
    Note over L: aggregation and significance test
{% end %}

Several evaluation criteria for a good interleaving method have been proposed in the literature,
for example in [Fidelity, Soundness, and Efficiency of Interleaved Comparison Methods](https://staff.fnwi.uva.nl/m.derijke/wp-content/papercite-data/pdf/hofmann-fidelity-2013.pdf):
* **fidelity**: the method declares the correct winner, in particular it does not prefer any ranker when clicks are random;
* **sensitivity** (or efficiency): the method needs as few observations as possible to reach a confident conclusion;
* **user experience**: the combined list is not worse than the lists being compared.

Each method has its own advantages and limitations with respect to these criteria.

## The bias of balanced interleaving

Balanced interleaving turns out to be biased
when the compared rankings are similar but shifted by a position or so,
as shown in [Large-Scale Validation and Analysis of Interleaved Search Evaluation](https://www.cs.cornell.edu/people/tj/publications/chapelle_etal_12a.pdf).

Let A = (a, b, c) and B = (b, c, a).
Depending on the coin, the combined list is either (a, b, c) or (b, a, c).
Now, assume a user who does not care about relevance at all and clicks on a single random result.

| Combined list | Clicked result | $k$ | Winner |
|---------------|----------------|-----|--------|
| (a, b, c)     | a              | 1   | A      |
| (a, b, c)     | b              | 1   | B      |
| (a, b, c)     | c              | 2   | B      |
| (b, a, c)     | b              | 1   | B      |
| (b, a, c)     | a              | 1   | A      |
| (b, a, c)     | c              | 2   | B      |

Ranker B wins two thirds of the queries, although the clicks carry no information whatsoever.
Ranker B just happens to rank c higher, and c ends up in the part of the list the credit attribution looks at.
Such a bias can easily flip the result of an experiment, since real rankers often differ by exactly such small shifts.

## What's next?

We have seen that interleaving compares two rankers on the same users and queries,
and that balanced interleaving gives a simple way to do it.
Unfortunately, it also prefers one of the rankers even when users click at random.

In the second part, which is coming soon, I will go through:
* Team-Draft interleaving, which fixes this bias and is the method most companies use in practice;
* the disadvantages of interleaving you should be aware of before replacing your A/B-tests;
* the sensitivity gains reported in the literature and by the industry;
* how Netflix, Airbnb, DoorDash, Etsy and others apply interleaving.

## Resources

- [Evaluating Retrieval Performance using Clickthrough Data | Joachims, 2002](https://www.cs.cornell.edu/~tj/publications/joachims_02b.pdf)
- [Large-Scale Validation and Analysis of Interleaved Search Evaluation | Chapelle et al., 2012](https://www.cs.cornell.edu/people/tj/publications/chapelle_etal_12a.pdf)
- [Fidelity, Soundness, and Efficiency of Interleaved Comparison Methods | Hofmann et al., 2013](https://staff.fnwi.uva.nl/m.derijke/wp-content/papercite-data/pdf/hofmann-fidelity-2013.pdf)
- [Effective Online Evaluation for Web Search | SIGIR 2019](https://dl.acm.org/doi/10.1145/3331184.3331378)
- [Online Evaluation for Effective Web Service Development | Yandex SIGIR 2019 Tutorial](https://research.yandex.com/tutorials/online-evaluation/sigir-2019)
- [A Short Survey on Online and Offline Methods for Search Quality Evaluation](https://link.springer.com/chapter/10.1007/978-3-319-41718-9_3)
- [Addressing Variance in AB tests: Interleaved Evaluation of Rankers | Erik Bernhardson, Haystack 2019](https://www.youtube.com/watch?v=-1npOZBQ7AQ)
- [Interleaving Experiments: Revolutionizing Recommender System Evaluation | Juan C Olamendy](https://medium.com/@juanc.olamendy/interleaving-experiments-revolutionizing-recommender-system-evaluation-3d42bc5e5ce2)
