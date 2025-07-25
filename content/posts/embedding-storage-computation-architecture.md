+++
title = "Designing an Embedding Storage and Vector Search System (1)"
date = "2024-12-01"
draft = true
description = "This post covers different considerations on how to build vector search and storage on top of a data lake."
[taxonomies]
tags=["embeddings", "vector-databases", "system-design"]
[extra]
comment = true
+++


## Introduction

Large Language Models are one of the biggest breakthrough in the AI field.
Since their discovery they have become a go-to solution in search engines, chatbots and recommender systems.
Also, plenty of systems leverage Retrieval-Augmented Generation (RAG) or the agentic approach.
This shift has drastically transformed the tools ML engineers use in their production environments.
One major challenge in LLM systems is usually overlooked though. 

Majority of the LLM systems use vector search which is not new. 
It has been widely adopted in the industry since the word2vec discovery. 
Not only in NLP but in other branches of machine learning too. 

The problem is how to choose a proper way of vector storage, computing and search.
Another issue is that it's hard to tune a specific tool for a particular task.
There are more than few ways to build a vector index so the number of hyperparameters scale greatly.

Nowadays, there is a wide spectrum of proprietary and open-source vector databases available.
The list grows every month.
Although they serve the common purpose the devil's in the details.
While some vector search engines such as pgvector offer seamless integration 
with your tech stack others like faiss focus on ease of use and performance.

Despite this, building a reliable solution has become more challenging than ever.
Integrating it into your infrastructure adds another layer of complexity.
Do you need only online vector search, or do you also need scheduled pipelines?
What about your data analytics and data scientists who would like to run ad-hoc queries?
It's important to keep these needs in mind. 
Naturally, here is no silver bullet. 
Some companies don't even need a vector database, naive search with numpy fits them.
Others need a high-load solution for billions of vectors updated every second. 

This post is an attempt to provide a framework for designing such systems
and list of points to consider.
I hope they'll help you in tailoring existing tools for your particular task.

## Taxonomy of Vector Databases

DBMS commonly divided into client-server and embedded.
Client-server is the type all are familiar with as this type is present in the majority of IT-systems. 
But the new (or not so) DBMS type has been emerging. Meet embedded databases. 
Actually, you already know one! SQLite is a great example. Both LanceDB and faiss are the examples of embedded vector databases.

Another classification of DBMS is a division into OLTP (Online Transactional Processing) and OLAP (Online Analytical Processing).
OLTP databases are meant to work under enormous load with requests being small queries. 
OLAP, on the other hand, are meant for big complex queries which can use lots of joins and other aggregations. 
PostgreSQL is an OLTP example whilst GreenPlum is an OLAP one.

Balancing between latency, recall, storage and overall cost can be achieved by
usage of different vector indexing algorithms.
The most popular is HNSW but the choice set is much bigger. 
Even more, there are other options available such as quantization to lower
the vectors' memory requirements and to speed up vector search further.

## Use Cases

- Vector Search Use Cases (online, batch, ad-hoc analytics: map applications to use cases e.g. online recsys)
- Recommender systems
- LLM Aplications
- BI & Analytics

## Choosing storage for Embeddings

It is a wide-used approach to divide a storage to multiple tiers based on 
the required read/write performance and the cost. There are often two tiers, hot and cold,
or three tiers, hot, warm and cold accordingly.

Hot storage tier is the most expensive but provides the best performance. 
This layer is usually dedicated to online data processing. 
PostgreSQL is a usual solution for a hot storage.

Warm storage tier is used for batch processing and OLAP scenarios. 
It tries to balance middle performance with middle costs. 
ClickHouse or Hadoop are both the warm storage examples. 
Any of the DFSs (Distributed File System) could serve a similar purpose.

Last but not least, cold storage's purpose is to store data which are not accessed on a regular basis.
Therefore, cold storage is the cheapest and the least performant one. 
A common example is an object storage (AWS S3, MinIO, Ceph).


## Do you need a new data format?

As we divide our storage to few tiers a new question rise. 
What data format should one use to store embeddings? 
Well, let's start with our needs. I, personally, would like to have a format which is:
- Fast enough for queries including vector search;
- Compatible with the current processing tools (Spark, Pandas, Polars);

Lance is an open-source column-based data storage format designed with ML applications in mind. 
It is based on parquet specification and tightly connected to the Apache Arrow ecosystem. 
That means that Lance files can be read with all the tools you already use for processing parquet.
You can load Lance file with pandas and polars as well.

Lance can store the whole datasets in one file. Audio, video and image data can be stored in a binary format.

The main advantage of Lance is embedding storage and search optimization.

Besides Lance there are a few formats which are arrow-compatible such as Iceberg and Delta Lake. 
They might be excellent solutions to the general data engineering problem, but their purpose is not
to make the life of ML Engineers easier. 

Another tools: apache orc, feather (apache arrow), tiledb, zarr



## Basics of Big Data Architecture and Why Does It Matter?

System layers: layers, highlight storage and processing,

Lambda architecture: how lambda affect embeddings

Kappa architecture: how kappa affect embeddings



## A real-world solution to the problem

## Additional Considerations

You should base your design on the current design of your data architecture. 
Additionally, ease of migration to a new system and ease of learning can be very important.

The post doesn't cover all requirements you should consider. Things not covered in the post:
data lifecycle management, gdpr compliance, security (access separation, rbac model), cloud vs on-premise.

You can also think about connecting the embedding storage and search systems with
your current analytics stack. 
E.g. you might be using Trino or Presto for ad-hoc queries and there is a need for an integration.

Additional to security considerations if you have enormous amount of embeddings it can be helpful to 
develop a metastore for embedding discovery inside your party.

- quokka (write oltp data to olap for next analysis)

Finally you need to consider:
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
- [Databricks Glossary: Lambda Architecture](https://www.databricks.com/glossary/lambda-architecture)
- [Bartosz Konieczny: Zeta Architecture](https://www.waitingforcode.com/general-big-data/zeta-architecture/read)
- [Databricks: "Delta vs. Lambda: Why Simplicity Trumps Complexity for Data Pipelines"](https://www.databricks.com/blog/2020/11/20/delta-vs-lambda-why-simplicity-trumps-complexity-for-data-pipelines.html)

- [IBM: What is Data Lifecycle Management](https://www.ibm.com/topics/data-lifecycle-management)
- https://github.com/quamernasim/Role-Based-Access-Control-of-Qdrant-Vector-Database
- [Cisco: Securing Vector Databases](https://sec.cloudapps.cisco.com/security/center/resources/securing-vector-databases)