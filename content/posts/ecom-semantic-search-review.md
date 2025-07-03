+++
title = "Survey on E-com Embedding Based Non-Personalized Retrieval Approaches"
date = "2025-02-20"
draft = true
description = "This post will cover basics of algorithms used to power semantic search of e-com platforms."
[taxonomies]
tags=["search", "information-retrieval"]

[extra]
comment = true
+++

## Outline

## Introduction
    * why use lexical search
    * drawbacks of lexical search
    * semantic search as a complementary component
    * drawbacks of semantic search
    * so, use hybrid models!
## Search engine architecture

* pre-ranking
* ranking

types of pre-ranking including embedding-based retrieval

## Introduction to embedding-based retrieval

(1) Latent Factor Models: Nonlinear matrix completion approaches that learn query and document-level embeddings
without using their content.
(2) Factorized Models: Separately convert queries and documents to low-dimensional embeddings based on content.
(3) Interaction Models: Build interaction matrices between
the query and document text and use neural networks to
mine patterns from the interaction matrix


* Model architectures
    * amazon
    * alibaba (taobao)
    * jd
    * instacart
    * walmart
    * etsy
* Ending?
* Resources


## Introduction

Go through sigir conferences

Summary of Semantic Retrieval Approaches



## Resources

* [Towards Personalized and Semantic Retrieval: An End-to-End Solution for E-commerce Search via Embedding Learning](https://arxiv.org/pdf/2006.02282)

* [Contrastive Representation learning](https://lilianweng.github.io/posts/2021-05-31-contrastive/)

* [Embedding based retrieval for search and recommendation](https://medium.com/better-ml/embedding-learning-for-retrieval-29af1c9a1e65)

* [An introduction to embedding-based retrieval](https://www.yuan-meng.com/posts/ebr/)

* [Evaluating embedding based retrieval beyond historical search results](https://haystackconf.com/eu2023/talk-13/)

* 27 Oct 2013 [Learning Deep Structured Semantic Models
 for Web Search using Clickthrough Data](https://posenhuang.github.io/papers/cikm2013_DSSM_fullversion.pdf)


* 2 Feb 2018 [User Intent, Behaviour, and Perceived Satisfaction in Product Search](https://dl.acm.org/doi/abs/10.1145/3159652.3159714)

* 11 Oct 2018 [BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding](https://arxiv.org/abs/1810.04805)

* 2016 [Amazon Search: The Joy of Ranking Products](https://www.amazon.science/publications/amazon-search-the-joy-of-ranking-products)

* 1 jul 2019 [Semantic Product Search](https://arxiv.org/pdf/1907.00937)

* 2019 [Share on Neural Product Retrieval at Walmart.com](https://dl.acm.org/doi/abs/10.1145/3308560.3316603)

* 17 Jun 2021 [Alibaba: Embedding-based product retrieval in Taobao Search ](https://github.com/liyinxiao/Ranking_Papers/blob/master/Alibaba/Embedding-based%20Product%20Retrieval%20in%20Taobao%20Search.pdf)

* 12 Sept 2022 [An Embedding-Based Grocery Search Model at Instacart](https://sigir-ecom.github.io/ecom22Papers/paper_8392.pdf)

* 9 Oct 2022 [Multi-Objective Personalized Product Retrieval in Taobao Search](https://arxiv.org/abs/2210.04170)

* 21 Feb 2023 [Que2Engage: Embedding-based Retrieval for Relevant and Engaging Products at Facebook Marketplace](https://arxiv.org/abs/2302.11052)

* 24 May 2023 [JDsearch: A Personalized Product Search Dataset with Real Queries and Full Interactions](https://arxiv.org/abs/2305.14810)

* 7 Jun 2023 [Unified Embedding Based Personalized Retrieval in
Etsy Search](https://arxiv.org/pdf/2306.04833)

* 22 Jul 2023 [XWalk: Random Walk Based Candidate Retrieval for Product Search](https://arxiv.org/pdf/2307.12019)

* 2023 [Web-scale semantic product search with large language models](https://www.amazon.science/publications/web-scale-semantic-product-search-with-large-language-models)

* 9 May 2024 [Optimizing E-commerce Search: Toward a Generalizable and Rank-Consistent Pre-Ranking Model](https://arxiv.org/pdf/2405.05606)

* 14 aug 2024 [Enhancing Relevance of Embedding-based Retrieval at Walmart](https://arxiv.org/html/2408.04884v2)

* 8 Oct 2024 [Embedding based retrieval for long tail search queries in ecommerce](https://dl.acm.org/doi/abs/10.1145/3640457.3688039)

* 5 dec 2024 [Semantic Retrieval at Walmart](https://arxiv.org/pdf/2412.04637)

* 13 Jan 2025 [Multimodal semantic retrieval for product search](https://arxiv.org/abs/2501.07365)

