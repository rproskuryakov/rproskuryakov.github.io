+++
title = "Interleaving, a Retrieval Online Evaluation Method Nobody Talks About"
date = "2025-07-30"
draft = true
description = "A/B-testing has become the main tool for almost any machine learning model online evaluation method. But there is another tool for search ranking and recommender systems in particular, interleaving. Somehow it is not mentioned enough despite its big advantages over A/B-testing in multiple cases. This post aims to close the gap."
[taxonomies]
tags=["search", "A/B-testing", "experimenting", "interleaving"]
[extra]
comment = true
+++

## Outline

Motivation for invention of A/B-testing (duration, biases, variance). 

Introduction of interleaving for variance reduction (and sensitivity increase) 
in online estimation of ranking quality.
Simple example with bar and light beer and dark beer. 

Introduction to balanced interleaving. 
Sample statistics. 
Paired t-test, binomial sign test.

Sensitivity gain. Biases of balanced interleaving. 

General framework for interleaving. Credit attribution and interleaving policy. 

Introduction to Team-Draft interleaving. Test statistics. Bootstrap. 

Drawbacks of Team-Draft.


## Outline of second post

Evolution of balanced interleaving to unbiased balanced interleaving.

Evolution of team-draft to general team-draft interleaving.

Final comparison of interleaving methods.

Application in industrial scenario. 
Usage of interleaving for testing a series of 
small-impact changes to pack them up into A/B-test in Netflix and DoorDash.


## Who is it for?


## Introduction

A/B-testing has become a widely adopted tool for online evaluation of machine learning models
across the industry in the past 15 years. Despite its advantages, there are still lots of issues to tackle.
The main culprit of any A/B-test is variance. Multiple efforts has been made throughout recent years to mitigate the problem.
For instance, the methods such as [CUPED](source link) were developed. 

Lets look at the variance sources in testing a search engine.

[Beyond A/B Testing: Part 2 – When A/B Tests Struggle with Ranking & Recommendations](https://bananimohapatra.substack.com/p/beyond-ab-testing-part-2-when-ab?utm_source=substack&utm_medium=email&utm_content=share)

What randomization unit?

First one: different users bring different contribution. Cola Example

Second one: different queries different algorithms. Too much uncertainty because of search queries nature. 

Cola Example

There is another method to reduce variance in such cases almost nobody talks about, interleaving. 
Originally, it was developed to test search engines, but it can be applied to any online tests of ranking models.

The idea behind interleaving was introduced by Thorsten Joachims in 2002. 

He introduced a method of combining two ranker results and proved that $R_a$ and $R_b$ 
being the relevance of rankers A and B correspondingly can be evaluated
by estimation of expectation $E(\frac{C_a - C_b}{C})$.
If this value is proven to be bigger than zero,  
then it can be said that the ranker A produces more relevant results than ranker B.
Such a hypothesis can be tested by using two-tailer paired t-test for samples $c_{a,i}/c_{i}$ and $c_{b,i}/c_{i}$.

Bad model affecting users.

## Interleaving Basics

Several evaluation criteria for a good interleaving method have
been proposed in the literature: correctness of the declared winning
ranker, sensitivity to improvements in search quality, and preserving user experience quality as an online intervention.

Interleaving methods distinguish from each other by how the interleaved result is constructed from the compared rankings (interleaving policy), and how user actions are attributed back to the
compared rankers (credit attribution). Each method has its own
advantages and limitations with respect to these criteria

### Origins and balanced interleaving

Formally, let $A = (a_1, a_2, ..., a_n)$ and $B = (b_1, b_2, ..., b_n)$ be outputs of two different ranking models.
Let $I = (i_1, i_2, ..., i_n)$ be the combined ranking computed by an interleaving policy. 
Let $c_1, c_2, ..., c_k$ be the ranks of the clicked items. 
To derive preference between A and B one compares the number of clicks in the top
$k = \min\{{j:(i_{c_{max}} = a_j) \or (i_{c_{max}} = b_j)\}}$

In practice, collecting samples that hold under the assumption of t-test so expectations of samples are distributed
normally can be challenging. In such cases, the author proposes an alternative 
approach, using the binomial signed test on median $M\frac{C_a - C_b}{C}$.

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

Latter contributors to the topic introduced the following metric:

$$ \Delta_{AB} = \frac{W_A + \frac{1}{2}T_{AB}}{W_A + W_B + T_{AB}} - 0.5$$

Positive value of $\Delta_{AB}$ indicates thet A > B and negative that B > A.

E.g.

Then the bootstrap method is used to estimate mean of the distribution. 
Subsamples are formed in such way that queries or sessions are drawn with replacement to each subsample.

One can also use boostrap percentile method to estimate the metric's confidence interval.

## Taxonomy

Generally speaking, interleaving methods strive to increase experiment metric sensitivity by leveraging one or both of the following:
credit attribution and interleaving policy. 

The simplest interleaving method, balanced interleaving [10,11], is biased
when comparing lists that are similar but up to small shifts in position [7,21].

The unbalancedness of balanced interleaving (example)

heads (a, b, d, c), tails (d, a, b, c) -> 0.5 - heads -> (a, d, b, c) (too similar to first one)


### Team-draft introduction
Team-Draft Interleaving was introduced in by Filip Radlinski, Madhu Kurup and Thorsten Joachims 
to partially compensate for drawbacks of balanced interleaving. 

[How Does Clickthrough Data Reflect Retrieval Quality?](https://www.cs.cornell.edu/people/tj/publications/radlinski_etal_08b.pdf)

image

Team-Draft interleaving drawbacks (still has biases). 
How does the biases affect the quality? If they are random enough, it's okay. 
They can compensate each other.

[Debiased Balanced Interleaving at Amazon Search](https://assets.amazon.science/a9/c8/c9016a1c47caac6a634768e7491d/debiased-balanced-interleaving-at-amazon-search.pdf)

Counterfactual evaluation framework for credit attribution.

Amazon claims to observe a 60x gain in sensitivity of interleaving over A/B tests.


[Fidelity, Soundness, and Efficiency of
Interleaved Comparison Methods](https://staff.fnwi.uva.nl/m.derijke/wp-content/papercite-data/pdf/hofmann-fidelity-2013.pdf)


weak transitivity property for comparing multiple rankers ()

treatment effect mapping (linear weighted least squared model) (sources )

linear model cant work with sign flips 
probability of the sign disagreement between ab and interleaving
[Interleaved Online Testing in Large Scale Systems | Amazon Search | DM4IR&Recsys (WWW'23)](https://www.youtube.com/watch?v=qFC5AGT62xw)

[Debiased Balanced Interleaving at Amazon Search](https://assets.amazon.science/a9/c8/c9016a1c47caac6a634768e7491d/debiased-balanced-interleaving-at-amazon-search.pdf)

Counterfactual evaluation framework for credit attribution.

Amazon claims to observe a 60x gain in sensitivity of interleaving over A/B tests.


[Fidelity, Soundness, and Efficiency of
Interleaved Comparison Methods](https://staff.fnwi.uva.nl/m.derijke/wp-content/papercite-data/pdf/hofmann-fidelity-2013.pdf)


weak transitivity property for comparing multiple rankers ()

treatment effect mapping (linear weighted least squared model) (sources )

linear model cant work with sign flips 
probability of the sign disagreement between ab and interleaving
[Interleaved Online Testing in Large Scale Systems | Amazon Search | DM4IR&Recsys (WWW'23)](https://www.youtube.com/watch?v=qFC5AGT62xw)



[Team-Draft Interleaving, AIC Analytics Day | Roman Poborchy](https://www.youtube.com/watch?v=voY7waRb_D0)

Remove probabilistic and team-draft interleavings

[//]: # (### Probabilistic interleaving)

[//]: # (Probabilistic Interleaving )

[//]: # ()
[//]: # ([A Probabilistic Method for)

[//]: # (Inferring Preferences from Clicks]&#40;https://www.cs.ox.ac.uk/people/shimon.whiteson/pubs/hofmanncikm11.pdf&#41;)

[//]: # ()
[//]: # (image)

[//]: # ()
[//]: # (Probabilistic interleaving drawbacks)

[//]: # ()
[//]: # (### Optimized interleaving)

[//]: # (Optimized interleaving)

[//]: # ()
[//]: # ([Optimized Interleaving for Online Retrieval Evaluation]&#40;https://www.microsoft.com/en-us/research/wp-content/uploads/2013/02/Radlinski_Optimized_WSDM2013.pdf.pdf&#41;)

[//]: # ()
[//]: # (image)

[//]: # ()
[//]: # (Optimized interleaving drawbacks)

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

Доклад wikimedia 

https://www.cs.cornell.edu/people/tj/publications/chapelle_etal_12a.pdf


## Resources


[Large-Scale Validation and Analysis of Interleaved Search Evaluation | Yahoo](https://github.com/wzhe06/Reco-papers/blob/master/Evaluation/%5BInterLeaving%5D%20Large-Scale%20Validation%20and%20Analysis%20of%20Interleaved%20Search%20Evaluation%20(Yahoo%202012).pdf)

[Interleaving Python Package](https://github.com/mpkato/interleaving)

[A Short Survey on Online and Offline Methods for Search Quality Evaluation](https://staff.fnwi.uva.nl/e.kanoulas/wp-content/uploads/A-Short-Survey-on-Evaluation.pdf)


[Online Testing for Learning-to-Rank Interleaving](https://sease.io/2020/05/online-testing-for-learning-to-rank-interleaving.html)

[Online Testing Learning to Rank with Solr Interleaving | Alessandro Benedetti](https://www.youtube.com/watch?v=iC5ffoInung)

[Addressing Variance in AB tests: Interleaved Evaluation of Rankers | Erik Bernhardson](https://www.youtube.com/watch?v=-1npOZBQ7AQ)

[Second link](https://haystackconf.com/2019/variance/)

[Interleaving Experiments: Revolutionizing Recommender System Evaluation | Juan C Olamendy](https://medium.com/@juanc.olamendy/interleaving-experiments-revolutionizing-recommender-system-evaluation-3d42bc5e5ce2)


