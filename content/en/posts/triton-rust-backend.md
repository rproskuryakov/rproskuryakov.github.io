---
author: rodion.proskuriakov
date: "2024-07-15"
description: Guide to deploying a retrieval model with the Triton Inference Server
tags:
- triton
- inference
- rust
title: Building a Custom Backend for the Triton Inference Server on Rust
comments: true
draft: true
---

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

