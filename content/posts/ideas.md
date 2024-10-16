+++
title = "Ideas"
date = "2019-03-09"
draft = true
#[taxonomies]
#tags=["homomorphic-encryption", "security"]
[extra]
comment = false
+++

# Homomorphic Encryption


Inference on data encrypted with Homomorphic Encryption


https://github.com/OpenMined/PySyft
https://github.com/microsoft/SEAL
https://github.com/FederatedAI/FATE
https://github.com/homenc/HElib
https://github.com/Microsoft/CryptoNets
https://crypten.ai/


Converting a model for Triton Inference Server to handle homomorphically encrypted data is a complex task that involves several interconnected challenges:
* Homomorphic Encryption Scheme: Choosing the right HE scheme (e.g., Paillier, BFV, CKKS) based on data type, noise tolerance, and performance requirements.
* Model Architecture: Analyzing the model architecture to identify compatible layers and operations. Many operations (e.g., ReLU, max pooling) are not directly supported by HE.
* Model Conversion: Modifying the model to use HE-compatible operations or approximating unsupported operations.
* Triton Integration: Integrating the modified model into Triton Inference Server, potentially requiring custom backend implementations.
* Performance Optimization: Addressing the significant performance overhead introduced by HE.
Potential Approaches
While there's no straightforward solution, here are some potential approaches:
1. HE-Friendly Model Architecture
* Restrict Model Complexity: Consider simpler models like linear regression or logistic regression, which are more amenable to HE.
* Approximate Non-Linear Operations: Use polynomial approximations or piecewise linear approximations for non-linear activations like ReLU.
* Quantization: Reduce precision to mitigate noise growth in HE computations.
2. Custom Backend for Triton
* Develop a Custom Backend: Create a Triton backend that handles encrypted data, performs HE computations, and returns encrypted results.
* Leverage HE Libraries: Utilize existing HE libraries (e.g., SEAL, OpenFHE) to implement the backend.
* Optimize Performance: Explore techniques like batching and parallelization to improve performance.
3. Hybrid Approach
* Partial Encryption: Encrypt only sensitive data and perform computations on cleartext for other parts of the model.
* Secure Aggregation: Aggregate encrypted data from multiple clients before decryption for model training or inference.
* Homomorphic Encryption for Specific Layers: Apply HE to specific layers where privacy is critical, while using cleartext for other layers.



Homomorphic Encryption (HE) and Differential Privacy (DP) are two powerful tools for safeguarding sensitive data, and they often complement each other in various applications. While they serve distinct purposes, their combination can offer robust privacy protection.
Homomorphic Encryption
HE allows computations to be performed directly on encrypted data without decrypting it first.

1 This ensures data confidentiality as it remains protected throughout the computation process. 2

Differential Privacy
DP adds noise to data to prevent inference about specific individuals from the released data.

It ensures that the presence or absence of an individual's data in the dataset has a minimal impact on the result.

Federated Learning: In federated learning, HE can protect the privacy of individual models, while DP can be used to add noise to model updates before sharing them with a central server.

# GPU Architecture

# Basics of GPU architecture and Nvidia specificity

https://www.cherryservers.com/blog/everything-you-need-to-know-about-gpu-architecture#:~:text=GPU%20architecture%20is%20everything%20that,functionality%20and%20efficiency%20of%20GPUs.

https://docs.nvidia.com/deeplearning/performance/dl-performance-gpu-background/index.html

## Flynn's taxonomy applied to GPU

## GPU Architecture Fundamentals

## GPU Execution Model

## What is CUDA?

## Performance Consideration

## Example of Performance Calculation

# LLM Uncertainty


Aleatoric and epistemic uncertainties + semantic uncertainty

https://github.com/zlin7/UQ-NLG

https://openreview.net/pdf?id=DZhqSyXAgo


[Concepts for Reliability of LLMs in Production](https://mlops.community/concepts-for-reliability-of-llms-in-production/)

https://arxiv.org/abs/2308.16175

[Semantic Uncertainty: Linguistic Invariances for Uncertainty Estimation in Natural Language Generation](https://arxiv.org/abs/2302.09664)

# History of Industrial Nvidia GPUs


# History of Nvidia Professional GPUs Microarchitectures

This is the second post in the GPU Architecture Series. The first post covering basics 
of GPUs architectures overall and particular features of Nvidia GPUs
for deep learning applications. This post will tell you the story behind Nvidia professional
GPUs. We will see what microarchitectures Nvidia invented, how they are related to one another, 
and what optimizations each microarchitectures has compared to the previous generation.


## Pascal microarchitecture

## Volta microarchitecture

## Ampere & Ampere Sparse microarchitecture

## Hopper microarchitecture

## Blackwell microarchitecture

## Links


# TRITON RUST BACKEND


Firstly you need to [install Rust](https://www.rust-lang.org/tools/install).

[Bindgen package: generation of FFI by C/C++ header file](https://docs.rs/bindgen/latest/bindgen/).

[Bindgen docs](https://rust-lang.github.io/rust-bindgen/)



```toml
[package]
name = "my_shared_library"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]
```

```cargo build --lib```
This will generate the shared library in the `target/debug` directory (or `target/release` if you use cargo build --release).

Symbol Visibility: For more complex scenarios, you might need to control symbol visibility using #[no_mangle] and extern "C" attributes.





Speeding up Triton Python Backend via Rust

rust cpu preprocessor 
https://docs.rs/tokenizers/latest/tokenizers/
https://github.com/xtuc/triton-rs/tree/main/example-backend 
https://docs.nvidia.com/deeplearning/triton-inference-server/archives/triton_inference_server_220/user-guide/docs/backend.html#backend-shared-library

https://github.com/triton-inference-server/core/blob/main/include/triton/core/tritonbackend.h
https://blog.rust-lang.org/2015/04/24/Rust-Once-Run-Everywhere.html
https://docs.rs/bindgen/latest/bindgen/struct.Builder.html
https://blog.asleson.org/2021/02/23/how-to-writing-a-c-shared-library-in-rust/
https://rust-lang.github.io/rust-bindgen/tutorial-0.html

pytorch postprocessing on GPU


cargo fmt, clippy и test
