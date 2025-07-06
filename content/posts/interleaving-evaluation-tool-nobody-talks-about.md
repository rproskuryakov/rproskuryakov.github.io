+++
title = "Interleaving, a Retrieval Online Evaluation Method Nobody Talks About"
date = "2025-07-30"
draft = true
description = "This post covers basics of interleaving and its relation to A/B-testing including different industrial use cases."
[taxonomies]
tags=["search", "A/B-testing", "experimenting", "interleaving"]
[extra]
comment = true
+++


## Introduction

A/B-testing has become the main tool for almost any machine learning model online evaluation method. 
But there is another tool for search ranking and recommender systems in particular, interleaving. 
Somehow it is not mentioned enough despite its big advantages over A/B-testing in multiple cases.

## Motivation

Cola Example

[Beyond A/B Testing: Part 2 – When A/B Tests Struggle with Ranking & Recommendations](https://bananimohapatra.substack.com/p/beyond-ab-testing-part-2-when-ab?utm_source=substack&utm_medium=email&utm_content=share)

## A/B-test disadvantages 

- Variance 
- Multiple groups
- Bad model affecting users

## Basics of interleaving

Several evaluation criteria for a good interleaving method have
been proposed in the literature: correctness of the declared winning
ranker, sensitivity to improvements in search quality, and preserving user experience quality as an online intervention.

Interleaving methods distinguish from each other by how the interleaved result is constructed from the compared rankings (interleaving policy), and how user actions are attributed back to the
compared rankers (credit attribution). Each method has its own
advantages and limitations with respect to these criteria

### Origins and balanced interleaving

The idea behind interleaving was introduced by Thorsten Joachims in 2002. 

He introduced a method of combining two ranker results and proved that $R_a$ and $R_b$ 
being the relevance of rankers A and B correspondingly can be evaluated
by estimation of expectation $E(\frac{C_a - C_b}{C})$.
If this value is proven to be bigger than zero,  
then it can be said that the ranker A produces more relevant results than ranker B.
Such a hypothesis can be tested by using two-tailer paired t-test for samples $c_{a,i}/c_{i}$ and $c_{b,i}/c_{i}$.

In practice, collecting samples that hold under the assumption of t-test so expectations of samples are distributed
normally can be challenging. In such cases, the author proposes an alternative 
approach, using the binomial signed test on median $M\frac_{C_a - C_b}{C}$. 

Joachims also provides an algorithm to produce the combined ranking. 

image (explanation)

Blind unbiased testing. The experiment data of the articles supports multiple states about the method. 
Firstly, the method evaluation results agree with manual relevance judgements. 
Secondly, the data supports two of the method's assumptions. 
The user clicks on more relevant links that non-relevant links on average.
Users do not click more frequently on links from one retrieval strategy independent 
of the relevance of the links.

[Evaluating Retrieval Performance using Clickthrough data](https://www.cs.cornell.edu/~tj/publications/joachims_02b.pdf)

Thus, balanced interleaving has emerged as a first iteration of the method. 

The simplest interleaving method, balanced interleaving [10,11], is biased
when comparing lists that are similar but up to small shifts in position [7,21].

[Fidelity, Soundness, and Efficiency of
Interleaved Comparison Methods](https://staff.fnwi.uva.nl/m.derijke/wp-content/papercite-data/pdf/hofmann-fidelity-2013.pdf)

[Debiased Balanced Interleaving at Amazon Search](https://assets.amazon.science/a9/c8/c9016a1c47caac6a634768e7491d/debiased-balanced-interleaving-at-amazon-search.pdf)

Counterfactual evaluation framework for credit attribution.

Amazon claims to observe a 60x gain in sensitivity of interleaving over A/B tests.

### Team-draft introduction
Team-Draft Interleaving was introduced in by Filip Radlinski, Madhu Kurup and Thorsten Joachims. 

[How Does Clickthrough Data Reflect Retrieval Quality?](https://www.cs.cornell.edu/people/tj/publications/radlinski_etal_08b.pdf)

image

Team-Draft interleaving drawbacks

[Team-Draft Interleaving, AIC Analytics Day | Roman Poborchy](https://www.youtube.com/watch?v=voY7waRb_D0)

### Probabilistic interleaving
Probabilistic Interleaving 

[A Probabilistic Method for
Inferring Preferences from Clicks](https://www.cs.ox.ac.uk/people/shimon.whiteson/pubs/hofmanncikm11.pdf)

image

Probabilistic interleaving drawbacks

### Optimized interleaving
Optimized interleaving

[Optimized Interleaving for Online Retrieval Evaluation](https://www.microsoft.com/en-us/research/wp-content/uploads/2013/02/Radlinski_Optimized_WSDM2013.pdf.pdf)

image

Optimized interleaving drawbacks

### Disadvantages of interleaving

Disadvantages of interleaving: preference-based metrics, hard to implement

###  Sensitivity comparison

[Optimized Interleaving for Online Retrieval Evaluation](https://www.microsoft.com/en-us/research/wp-content/uploads/2013/02/Radlinski_Optimized_WSDM2013.pdf.pdf)

[Comparing the Sensitivity of Information Retrieval Metrics](https://www.microsoft.com/en-us/research/wp-content/uploads/2010/07/fp146-radlinski.pdf)

[Effective Online Evaluation for Web Search](https://dl.acm.org/doi/10.1145/3331184.3331378)

https://research.yandex.com/tutorials/online-evaluation/sigir-2019

[AWS Samples Workshop](https://github.com/aws-samples/retail-demo-store/blob/master/workshop/3-Experimentation/3.3-Interleaving-Experiment.ipynb)


## Industrial Applications

Meta 

[How to Evaluate Ranking Algorithm Performance | Nancy Cheng | Meta](https://www.youtube.com/watch?v=6NbHLwaeY6E)


Netflix

[Innovating Faster on Personalization Algorithms at Netflix Using Interleaving | Netflix](https://netflixtechblog.com/interleaving-in-online-experiments-at-netflix-a04ee392ec55)


DoorDash

[How DoorDash is pushing experimentation boundaries with interleaving designs | DoorDash | August 27, 2024 | Tim Knapik & Stas Sajin](https://careersatdoordash.com/blog/doordash-experimentation-with-interleaving-designs/)

Etsy

[Faster ML Experimentation at Etsy with Interleaving | Etsy](https://www.etsy.com/codeascraft/faster-ml-experimentation-at-etsy-with-interleaving)


AirBnB

[Beyond A/B Test: Speeding up Airbnb Search Ranking Experimentation through Interleaving](https://medium.com/airbnb-engineering/beyond-a-b-test-speeding-up-airbnb-search-ranking-experimentation-through-interleaving-7087afa09c8e)


[//]: # (Amazon)

[//]: # (Yahoo)

[//]: # (Yandex)

## Conclusion

## Resources

[Interleaved Online Testing in Large Scale Systems | Amazon Search | DM4IR&Recsys (WWW'23)](https://www.youtube.com/watch?v=qFC5AGT62xw)

[Large-Scale Validation and Analysis of Interleaved Search Evaluation | Yahoo](https://github.com/wzhe06/Reco-papers/blob/master/Evaluation/%5BInterLeaving%5D%20Large-Scale%20Validation%20and%20Analysis%20of%20Interleaved%20Search%20Evaluation%20(Yahoo%202012).pdf)

[Interleaving Python Package](https://github.com/mpkato/interleaving)

[A Short Survey on Online and Offline Methods for Search Quality Evaluation](https://staff.fnwi.uva.nl/e.kanoulas/wp-content/uploads/A-Short-Survey-on-Evaluation.pdf)


[Online Testing for Learning-to-Rank Interleaving](https://sease.io/2020/05/online-testing-for-learning-to-rank-interleaving.html)

[Online Testing Learning to Rank with Solr Interleaving | Alessandro Benedetti](https://www.youtube.com/watch?v=iC5ffoInung)

[Addressing Variance in AB tests: Interleaved Evaluation of Rankers | Erik Bernhardson](https://www.youtube.com/watch?v=-1npOZBQ7AQ)

[Second link](https://haystackconf.com/2019/variance/)

[Interleaving Experiments: Revolutionizing Recommender System Evaluation | Juan C Olamendy](https://medium.com/@juanc.olamendy/interleaving-experiments-revolutionizing-recommender-system-evaluation-3d42bc5e5ce2)


