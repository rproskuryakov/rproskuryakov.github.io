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

## A/B-test disadvantages 

- Variance 
- Multiple groups
- Bad model affecting users

## Basics of interleaving

Balanced interleaving pg

image

Debiasing balanced interleaving pg (drawbacks)

Team-Draft Interleaving

image

Team-Draft interleaving drawbacks

Probabilistic Interleaving

image

Probabilistic interleaving drawbacks

Optimized interleaving

image

Optimized interleaving drawbacks

Disadvantages of interleaving: preference-based metrics, hard to implement
## Industrial Applications

Netflix

DoorDash

Amazon

Etsy

AirBnB

Yahoo

Yandex

## Conclusion

## Resources

[Optimized Interleaving for Online Retrieval Evaluation | Microsoft](https://www.microsoft.com/en-us/research/wp-content/uploads/2013/02/Radlinski_Optimized_WSDM2013.pdf.pdf)

[AWS Samples Workshop](https://github.com/aws-samples/retail-demo-store/blob/master/workshop/3-Experimentation/3.3-Interleaving-Experiment.ipynb)

[Interleaved Online Testing in Large Scale Systems | Amazon Search | DM4IR&Recsys (WWW'23)](https://www.youtube.com/watch?v=qFC5AGT62xw)

[Effective Online Evaluation for Web Search](https://dl.acm.org/doi/10.1145/3331184.3331378)

[Same but tutorial on sigir 19 | Yandex](https://research.yandex.com/tutorials/online-evaluation/sigir-2019)

[Evaluating Aggregated Search Using Interleaving | Yandex](https://staff.fnwi.uva.nl/m.derijke/content/publications/cikm2013-interleaving.pdf)

[Team-Draft Interleaving, AIC Analytics Day | Roman Poborchy](https://www.youtube.com/watch?v=voY7waRb_D0)

[How to Evaluate Ranking Algorithm Performance | Nancy Cheng | Meta](https://www.youtube.com/watch?v=6NbHLwaeY6E)

[Innovating Faster on Personalization Algorithms at Netflix Using Interleaving | Netflix](https://netflixtechblog.com/interleaving-in-online-experiments-at-netflix-a04ee392ec55)

[How DoorDash is pushing experimentation boundaries with interleaving designs | DoorDash | August 27, 2024 | Tim Knapik & Stas Sajin](https://careersatdoordash.com/blog/doordash-experimentation-with-interleaving-designs/)

[Faster ML Experimentation at Etsy with Interleaving | Etsy](https://www.etsy.com/codeascraft/faster-ml-experimentation-at-etsy-with-interleaving)

[Beyond A/B Test: Speeding up Airbnb Search Ranking Experimentation through Interleaving](https://medium.com/airbnb-engineering/beyond-a-b-test-speeding-up-airbnb-search-ranking-experimentation-through-interleaving-7087afa09c8e)

[Large-Scale Validation and Analysis of Interleaved Search Evaluation | Yahoo](https://github.com/wzhe06/Reco-papers/blob/master/Evaluation/%5BInterLeaving%5D%20Large-Scale%20Validation%20and%20Analysis%20of%20Interleaved%20Search%20Evaluation%20(Yahoo%202012).pdf)



[Interleaving Python Package](https://github.com/mpkato/interleaving)

[A Short Survey on Online and Offline Methods for Search Quality Evaluation](https://staff.fnwi.uva.nl/e.kanoulas/wp-content/uploads/A-Short-Survey-on-Evaluation.pdf)

[Online Evaluation A/B testing, Interleaving and Lerot](https://staff.fnwi.uva.nl/e.kanoulas/wp-content/uploads/Lecture-2-Online-Evaluation.pdf)

[Online Evaluation of Rankers Using Multileaving](https://di.ku.dk/english/research/phd/phd-theses/2018/Brian_Brost_Thesis.pdf)

[Evaluating the best AB testing metrics for search](https://www.algolia.com/blog/engineering/a-b-testing-metrics-evaluating-the-best-metrics-for-your-search)

[Online Testing for Learning-to-Rank Interleaving](https://sease.io/2020/05/online-testing-for-learning-to-rank-interleaving.html)

[Online Evaluation and LTR | Notes on AI | Paras Dahal](https://notesonai.com/online+evaluation+and+ltr)

[Online Testing Learning to Rank with Solr Interleaving | Alessandro Benedetti](https://www.youtube.com/watch?v=iC5ffoInung)

[Addressing Variance in AB tests: Interleaved Evaluation of Rankers | Erik Bernhardson](https://www.youtube.com/watch?v=-1npOZBQ7AQ)

[Second link](https://haystackconf.com/2019/variance/)

[The Joy of A/B-testing Part II Advanced Topics](https://medium.com/data-science/the-joy-of-a-b-testing-part-ii-advanced-topics-6c7f6cf71e4c_)

[Taking the Counterfactual Online: Efficient and Unbiased Online Evaluation for Ranking - ICTIR 2020](https://www.youtube.com/watch?v=sfKIjo3mYQ0)

[Interleaving Experiments: Revolutionizing Recommender System Evaluation | Juan C Olamendy](https://medium.com/@juanc.olamendy/interleaving-experiments-revolutionizing-recommender-system-evaluation-3d42bc5e5ce2)

[Beyond A/B Testing: Part 2 – When A/B Tests Struggle with Ranking & Recommendations](https://bananimohapatra.substack.com/p/beyond-ab-testing-part-2-when-ab?utm_source=substack&utm_medium=email&utm_content=share)

[Evaluating Retrieval Performance using Clickthrough data](https://www.cs.cornell.edu/~tj/publications/joachims_02b.pdf)

