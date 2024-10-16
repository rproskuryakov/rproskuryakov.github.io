+++
title = "Designing an Embedding Storage and Vector Search System"
date = "2024-12-01"
draft = true
[taxonomies]
tags=["embeddings", "vector-databases", "architecture"]
[extra]
comment = true
+++


## Introduction

- problem statement and needs for a new solutions

## Motivation

- Motivation (challenges e.g. scalability, performance bottlenecks, complexity of finding the right db)

## Vector Databases 201

vector databases (client-server and embedded; oltp vs olap; indexing strategies; tradeoffs: scaling, performance, ease of integration)

### Client-server vs Embedded

### OLTP vs OLAP

### Indexing strategies

### Tradeoffs

## Use Cases

- Vector Search Use Cases (online, batch, ad-hoc analytics: map applications to use cases e.g. online recsys)

### Recommender Systems

### LLM Applications

### BI & Analytics

## Choosing storage for Embeddings

- Choosing storage for Embeddings (multi-tier architecture: why to separate, what kind of data to store in each storage)

### Multi-tier Storage

## Do you need a new data format?
- Why do you need a new data format? (vector storage formats and why do you need one (lance vs parquet))
- storage formats: (lance, iceberg, data lake) apache orc, feather (apache arrow), tiledb, zarr

### Parquet

### Lance

## Basics of Big Data Architecture and Why Does It Matter?
- 
Basics of Big Data Architecture and Why does it matter? (big data architectures (layers, highlight storage and processing, lambda vs kappa: how they layout to embeddings))

### System Layers

### Lambda Architecture

### Kappa Architecture



## A real-world solution to the problem

## Additional Considerations

- Additional Consideration (security, dlm, gdpr, ccpa, cloud vs on-premise)
- feature processing and feature stores
- your current dwh architecture
- ease of delivery and migration as well as ease of learning
- metastore
- query engines (trino and sparksql)
- quokka (write oltp data to olap for next analysis)

requirements:
- vector's size
- application scenario (online/offline/near-offline) – data processing architectures
- your current dwh architecture
- query and writing frequency
- latency requirements: choose the right index
- olap analytics

## Conclusion


## Resources

- [Vector databases (4): Analyzing the trade-offs (Prashanth Rao)](https://thedataquarry.com/posts/vector-db-4/)
- [Qdrant](https://qdrant.tech/)
- [Faiss](https://github.com/facebookresearch/faiss)

- [Weaviate: Managing Resources](https://weaviate.io/developers/weaviate/starter-guides/managing-resources)
- [Parallels: What is Tiered Storage](https://www.parallels.com/blogs/ras/tiered-storage/?srsltid=AfmBOopU6wj50laXPu1zStAVL_MiC7dKSIl3jrmnC8c5sV1WTk9xdoaw)

Necessary:
- [Lance Data Format: Design Document](https://lancedb.github.io/lance/format.html)
- [LanceDB Blog: Benchmarking Random Access in Lance](https://blog.lancedb.com/benchmarking-random-access-in-lance/)
- [Lance GitHub](https://github.com/lancedb/lance)
- [DuckDB](https://duckdb.org/)

Optional:
- [Lance: Deep Dive](https://drive.google.com/file/d/1Orh9rK0Mpj9zN_gnQF1eJJFpAc6lStGm/view)
- [LanceDB GitHub](https://github.com/lancedb/lancedb)
- [LanceDB Blog: LanceDB + Polars](https://blog.lancedb.com/lancedb-polars-2d5eb32a8aa3/)
- [LanceDB: Universal Multimodal Data Lake with Lance and Trino](https://trino.io/assets/blog/trino-fest-2024/lance-characterai.pdf)
- [Embedded databases (1): The harmony of DuckDB, KùzuDB and LanceDB (Prashanth Rao)](https://thedataquarry.com/posts/embedded-db-1/)

Necessary:
- [Azure Architecture Center: Big Data Architectures](https://learn.microsoft.com/en-us/azure/architecture/databases/guide/big-data-architectures)
- [The Pipeline: Lambda, Kappa, Delta Architectures for Data](https://subrabytes.dev/dataarchitectures)
- [Milvus 2.0: Redefining Vector Database](https://milvus.io/blog/milvus2.0-redefining-vector-database.md)
- [Engineering at Meta: Scaling data ingestion for machine learning training at Meta](https://engineering.fb.com/2022/09/19/ml-applications/data-ingestion-machine-learning-training-meta/)

Optional:
- [Bartosz Konieczny: Data Engineering Design Patterns](https://www.oreilly.com/library/view/data-engineering-design/9781098165826/)
- [MongoDB: What is Big Data Architecture](https://www.mongodb.com/resources/basics/big-data-explained/architecture)
- [MongoDB: Data Lake Architecture](https://www.mongodb.com/resources/basics/databases/data-lake-architecture)
- [MinIO Blog: "The Architect’s Guide: A Modern Datalake Reference Architecture"](https://blog.min.io/the-architects-guide-a-modern-datalake-reference-architecture/)
- [Databricks: Lambda Architecture](https://www.databricks.com/glossary/lambda-architecture)
- [Bartosz Konieczny: Zeta Architecture](https://www.waitingforcode.com/general-big-data/zeta-architecture/read)
- [Databricks: "Delta vs. Lambda: Why Simplicity Trumps Complexity for Data Pipelines"](https://www.databricks.com/blog/2020/11/20/delta-vs-lambda-why-simplicity-trumps-complexity-for-data-pipelines.html)

- [IBM: What is Data Lifecycle Management](https://www.ibm.com/topics/data-lifecycle-management)
- https://github.com/quamernasim/Role-Based-Access-Control-of-Qdrant-Vector-Database
- [Cisco: Securing Vector Databases](https://sec.cloudapps.cisco.com/security/center/resources/securing-vector-databases)