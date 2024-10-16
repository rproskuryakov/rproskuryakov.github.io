+++
title = "Building a Custom Backend for the Triton Inference Server on Rust"
date = "2024-08-05"
comment = true
draft = true
[taxonomies]
tags=["triton", "inference", "rust"]
[extra]
comment=true
+++

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
