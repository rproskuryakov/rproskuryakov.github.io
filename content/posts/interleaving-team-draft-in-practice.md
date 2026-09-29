+++
title = "Interleaving, a Retrieval Online Evaluation Method Nobody Talks About. Part 2"
date = "2025-07-31"
draft = true
description = "The second part of the series on interleaving. It introduces Team-Draft interleaving, discusses the limitations of interleaving compared to A/B-testing, and shows how Netflix, Airbnb, DoorDash, Amazon and others use it to speed up ranking experiments."
[taxonomies]
tags=["search", "A/B-testing", "experimenting", "interleaving"]
[extra]
comment = true
mermaid = true
+++


## Introduction

In the [first part](@/posts/interleaving-evaluation-tool-nobody-talks-about.md) of the series,
we looked at why A/B-tests struggle to evaluate ranking models and how interleaving helps.
Instead of splitting users into two groups, interleaving shows every user a single list combining the results of both rankers
and uses the user's clicks to decide which ranker is better.

We also saw that any interleaving method consists of two parts:
* **interleaving policy**: how the combined list is constructed from the compared rankings;
* **credit attribution**: how user actions are attributed back to the compared rankers.

Finally, we found out that the first method, balanced interleaving, is biased.
For A = (a, b, c) and B = (b, c, a), a user who clicks at random makes B win two thirds of the queries.

## Overview

In this part, I will walk you through:
* Team-Draft interleaving, the method most companies use in practice, and its own drawbacks;
* the disadvantages of interleaving compared to A/B-tests;
* the sensitivity gains reported in the literature and by the industry;
* how Netflix, Airbnb, DoorDash, Etsy and others apply interleaving.

So, let's get started!

## Team-Draft interleaving

Team-Draft interleaving was introduced by Filip Radlinski, Madhu Kurup and Thorsten Joachims in
[How Does Clickthrough Data Reflect Retrieval Quality?](https://www.cs.cornell.edu/people/tj/publications/radlinski_etal_08b.pdf)
to compensate for the drawbacks of balanced interleaving.

### Interleaving policy

The method is inspired by how team captains pick players in a friendly football match.
In each round, a coin decides which captain picks first.
Each captain then picks their highest-ranked result that is not yet in the combined list, and that result joins their team.

{% mermaid() %}
flowchart TD
    Start(["Start: empty list, empty teams"]) --> Check{"List is full or<br/>rankings exhausted?"}
    Check -->|yes| Done(["Show list, log teams"])
    Check -->|no| Coin{"Coin flip"}
    Coin -->|"A first"| PA1["A adds its best result<br/>not yet in the list, team A"]
    PA1 --> PB1["B adds its best result<br/>not yet in the list, team B"]
    Coin -->|"B first"| PB2["B adds its best result<br/>not yet in the list, team B"]
    PB2 --> PA2["A adds its best result<br/>not yet in the list, team A"]
    PB1 --> Check
    PA2 --> Check
{% end %}

```python
import random


def team_draft_interleave(a: list, b: list, length: int) -> tuple[list, dict]:
    result, teams = [], {}
    rankings = {"A": a, "B": b}
    while len(result) < length:
        order = ["A", "B"] if random.random() < 0.5 else ["B", "A"]
        added = False
        for team in order:
            candidate = next((doc for doc in rankings[team] if doc not in teams), None)
            if candidate is not None and len(result) < length:
                result.append(candidate)
                teams[candidate] = team
                added = True
        if not added:
            break
    return result, teams
```

### Credit attribution

Credit attribution is straightforward: a click on a result counts for the team that picked it.
The ranker with more clicks wins the query, and the outcomes are aggregated with $\Delta_{AB}$ and the bootstrap
exactly as for [balanced interleaving](@/posts/interleaving-evaluation-tool-nobody-talks-about.md#statistics).

Let's return to the example that [broke balanced interleaving](@/posts/interleaving-evaluation-tool-nobody-talks-about.md#the-bias-of-balanced-interleaving), A = (a, b, c) and B = (b, c, a).
Whichever team picks first in a round, the coin makes it A or B with equal probability,
so both teams end up with the same number of results at every position in expectation.
A user who clicks randomly therefore gives the same expected credit to both teams,
and neither ranker is preferred.

### Drawbacks of Team-Draft

Team-Draft is not perfect either.
[Hofmann et al.](https://staff.fnwi.uva.nl/m.derijke/wp-content/papercite-data/pdf/hofmann-fidelity-2013.pdf) showed that it can fail to detect a real difference between rankers.
Let A = (a, b, c) and B = (b, c, a), and let c be the only relevant result.
B is clearly better, as it puts c at the second position instead of the third.
In the first round, A picks a and B picks b in some order.
In the second round, whichever team picks first takes c, and it is A or B with equal probability.
In expectation, the outcome is a tie.

How much do such biases matter in practice?
If they occur randomly and in both directions, they compensate each other over many queries.
Still, they reduce sensitivity and may lead to a wrong conclusion for particular pairs of rankers,
which motivated the methods we will look at in the third part.

## Disadvantages of interleaving

Before rushing to replace your A/B-tests, keep in mind the limitations of interleaving.

**Preference, not magnitude.**
Interleaving tells you *which* ranker users prefer, but not *by how much* your business metric will change.
You won't get a conversion or revenue lift from an interleaving experiment.
Amazon approaches this with treatment effect mapping: a linear model, fit on past experiments that were run both ways,
maps the interleaving preference to the expected A/B effect.
However, a linear mapping cannot handle cases where the interleaving and the A/B-test disagree on the sign,
so the probability of such disagreement has to be estimated as well,
see [Interleaved Online Testing in Large Scale Systems](https://www.youtube.com/watch?v=qFC5AGT62xw).

**Implementation complexity.**
You need a service that queries both rankers for each request, merges the results,
and logs which ranker every shown item came from.
Both rankers have to run in production at the same time, which doubles the ranking cost for the experiment traffic.
Latency of the slower ranker defines the latency of the response.

**Ranked lists only.**
Interleaving works only when the output is a list of items that can be mixed.
Changes to the UI, pricing or any effect that spans the whole page cannot be tested this way.

**Short-term signal.**
The credit is computed from immediate interactions such as clicks or add-to-carts.
Long-term effects, such as retention, remain the domain of A/B-tests.

## Sensitivity in practice

The main reason to deal with all of this is sensitivity.
[Comparing the Sensitivity of Information Retrieval Metrics](https://www.microsoft.com/en-us/research/wp-content/uploads/2010/07/fp146-radlinski.pdf)
and the large-scale validation at Yahoo by Chapelle et al.
showed that interleaving needs much less data than absolute click metrics to reach the same confidence,
while agreeing with editorial judgements on the direction of the difference.

The industry reports similar results:

| Company | Reported gain over A/B-testing        | Source                                                                                                                                                                  |
|---------|---------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Netflix | over 100x fewer users needed          | [Netflix Tech Blog](https://netflixtechblog.com/interleaving-in-online-experiments-at-netflix-a04ee392ec55)                                                              |
| Airbnb  | around 50x faster experiments         | [Airbnb Engineering](https://medium.com/airbnb-engineering/beyond-a-b-test-speeding-up-airbnb-search-ranking-experimentation-through-interleaving-7087afa09c8e)          |
| Amazon  | around 60x higher sensitivity         | [Debiased Balanced Interleaving at Amazon Search](https://www.amazon.science/publications/debiased-balanced-interleaving-at-amazon-search) |

The exact numbers depend on the product, traffic and metric, so treat them as an order of magnitude rather than a promise.

## Industrial Applications

Interleaving is applied in multiple big tech companies.
The most common pattern is a two-stage funnel.
Interleaving is used to quickly screen many small-impact hypotheses, such as a change in the query embedding algorithm or a new ranking feature.
Then only the winners, often bundled together, go to an A/B-test to estimate the business metric impact.

{% mermaid() %}
flowchart LR
    H["Many ranking hypotheses"] --> I["Interleaving:<br/>days, small traffic"]
    I -->|"losers"| X["Discarded"]
    I -->|"winners"| P["Bundle of winners"]
    P --> AB["A/B-test:<br/>business metrics"]
    AB --> R["Rollout"]
{% end %}

**Netflix** uses interleaving as the first stage of experimentation for personalization algorithms
and found that interleaving preferences strongly correlate with A/B-test metrics,
see [Innovating Faster on Personalization Algorithms at Netflix Using Interleaving](https://netflixtechblog.com/interleaving-in-online-experiments-at-netflix-a04ee392ec55).

**Airbnb** applies interleaving to search ranking and shares a lot of practical details,
such as how to handle competitive pairs and attribute bookings rather than clicks,
see [Beyond A/B Test: Speeding up Airbnb Search Ranking Experimentation through Interleaving](https://medium.com/airbnb-engineering/beyond-a-b-test-speeding-up-airbnb-search-ranking-experimentation-through-interleaving-7087afa09c8e).

**DoorDash** runs interleaving designs for store and item ranking and discusses variance estimation and its limitations,
see [How DoorDash is pushing experimentation boundaries with interleaving designs](https://careersatdoordash.com/blog/doordash-experimentation-with-interleaving-designs/).

**Etsy** uses interleaving for search ranking models,
see [Faster ML Experimentation at Etsy with Interleaving](https://www.etsy.com/codeascraft/faster-ml-experimentation-at-etsy-with-interleaving).

**Thumbtack** describes how interleaving accelerated ranking experimentation in a two-sided marketplace,
see [Accelerating Ranking Experimentation at Thumbtack with Interleaving](https://medium.com/thumbtack-engineering/accelerating-ranking-experimentation-at-thumbtack-with-interleaving-20cbe7837edf).

**Meta** covers interleaving among other approaches to evaluating ranking algorithms,
see [How to Evaluate Ranking Algorithm Performance](https://www.youtube.com/watch?v=6NbHLwaeY6E).

If you want to try interleaving yourself, the [interleaving](https://github.com/mpkato/interleaving) Python package
implements the most popular methods, and the [AWS Retail Demo Store workshop](https://github.com/aws-samples/retail-demo-store/blob/master/workshop/3-Experimentation/3.3-Interleaving-Experiment.ipynb)
walks through an interleaving experiment end to end.
Solr also ships interleaving support for Learning to Rank models out of the box.

## What's next?

This part covered Team-Draft interleaving, the limitations of interleaving and how the industry applies it.
In the third part, I will go through:
* the evolution of balanced interleaving into debiased balanced interleaving used at Amazon Search;
* the evolution of Team-Draft into generalized Team-Draft interleaving;
* probabilistic and optimized interleaving;
* comparing more than two rankers at once and the weak transitivity property;
* a final comparison of interleaving methods.

## Resources

- [How Does Clickthrough Data Reflect Retrieval Quality? | Radlinski, Kurup, Joachims, 2008](https://www.cs.cornell.edu/people/tj/publications/radlinski_etal_08b.pdf)
- [Comparing the Sensitivity of Information Retrieval Metrics | Radlinski, Craswell, 2010](https://www.microsoft.com/en-us/research/wp-content/uploads/2010/07/fp146-radlinski.pdf)
- [Large-Scale Validation and Analysis of Interleaved Search Evaluation | Chapelle et al., 2012](https://www.cs.cornell.edu/people/tj/publications/chapelle_etal_12a.pdf)
- [Fidelity, Soundness, and Efficiency of Interleaved Comparison Methods | Hofmann et al., 2013](https://staff.fnwi.uva.nl/m.derijke/wp-content/papercite-data/pdf/hofmann-fidelity-2013.pdf)
- [Optimized Interleaving for Online Retrieval Evaluation | Radlinski, Craswell, 2013](https://www.microsoft.com/en-us/research/wp-content/uploads/2013/02/Radlinski_Optimized_WSDM2013.pdf.pdf)
- [Debiased Balanced Interleaving at Amazon Search](https://www.amazon.science/publications/debiased-balanced-interleaving-at-amazon-search)
- [Interleaved Online Testing in Large Scale Systems | Amazon Search | DM4IR&Recsys (WWW'23)](https://www.youtube.com/watch?v=qFC5AGT62xw)
- [Team-Draft Interleaving, AIC Analytics Day | Roman Poborchy](https://www.youtube.com/watch?v=voY7waRb_D0)
- [Online Testing for Learning-to-Rank: Interleaving | Sease](https://sease.io/2020/05/online-testing-for-learning-to-rank-interleaving.html)
- [Online Testing Learning to Rank with Solr Interleaving | Alessandro Benedetti](https://www.youtube.com/watch?v=iC5ffoInung)
- [Interleaving Python Package](https://github.com/mpkato/interleaving)
- [AWS Retail Demo Store: Interleaving Experiment Workshop](https://github.com/aws-samples/retail-demo-store/blob/master/workshop/3-Experimentation/3.3-Interleaving-Experiment.ipynb)
