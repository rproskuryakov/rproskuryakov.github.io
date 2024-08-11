---
author: rodion.proskuriakov
date: "2024-08-05"
description: Как вывести в продакшн модель эмбеддингов с Triton Inference Server
tags:
- triton
- inference
- transformers
- onnxruntime
- tensorrt
title: Как вывести в продакшн модель эмбеддингов с Triton Inference Server
comments: true
draft: true
---

## Introduction  
  
I work in the MLOps team, and I recently got a  task   
to update a fine-tuned retrieval model running on Triton Inference Server.   
It had poor performance and the customer needed lower latency and higher RPS.  
The original deployment was on pure PyTorch so I needed to convert model to ONNX and do some optimizations.  
It was an interesting journey so i decided yo share it with others.
  
## Overview  
  
In this tutorial I will guide you through deploying `intfloat/multilingual-e5-large` model with Triton Inference Server.  
We will also look on how to convert it to ONNX format and optimize it with TensorRT Execution Provider for ONNX Runtime.  
For running the code I use the following server configuration: <SERVER_CONFIG>.  
All code can be found in [this repository](https://github.com/rproskuryakov/triton-sentence-transformer-tutorial).  
So let's dive in!  
  
## Benchmarking Methodology  
  
Therefore I will use [locust.io](https://locust.io/) for the purpose of this tutorial.  
  
## Baseline: Python Backend  
  
The simplest way to deploy a model with the Triton inference server is to pack it with the Python backend and compute via [huggingface pipeline](https://huggingface.co/docs/transformers/v4.44.0/en/main_classes/pipelines). All you should know now is that the python backend requires model implementation in the `<model_name>/1/model.py` file. The standard directory structure for inferencing a model on the Triton can be found [here](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/model_repository.html#repository-layout).  
  
Firstly you need to define a class named TritonPythonModel and implement execute method. Optionally you can also implement initialize and finalize methods if your model needs to load something before execution or do some clean ups before exiting.  
  
```python  
from transformers import pipeline  
import triton_python_backend_utils as pb_utils  
  
  
class TritonPythonModel:  
 """Your Python model must use the same class name. Every Python model that is created must have "TritonPythonModel" as the class name. """  
 def initialize(self, args):  
  self._pipeline = pipeline(  
 "feature-extraction", model="intfloat/multilingual-e5-large", )  
 def execute(self, requests):  
 responses = []  
 for request in requests: input_ = pb_utils.get_input_tensor_by_name(request, "text")  
 input_string = input_.as_numpy()  
 requests_texts = [i[0].decode("utf-8") for i in input_string]  
  
 batch = self.tokenizer(  
 requests_texts, max_length=512, padding=True, truncation=True, return_tensors="np", ) responses.append(  
 pb_utils.InferenceResponse(  
 output_tensors=[ pb_utils.Tensor(  
 "output_input_ids", batch["input_ids"], ), pb_utils.Tensor(  
 "output_attention_mask", batch["attention_mask"], ) ] ) )  
 return responses  
```  
  
In the model configuration file you need to define model name and chosen backend.  
  
```prototext  
name: "multilingual-e5-large"  
backend: "python"  
```  
  
You also need to define inputs and output of the model which include names, data types and dimensions.  
String type always has size one dimension.  
  
```prototext  
input [  
 { name: "text" data_type: TYPE_STRING dims: [ 1 ] }]  
output [  
 { name: "output_vector" data_type: TYPE_FP32 dims: [ 1024 ] }]  
```  
  
  
## What is ONNX  
  
The ONNX is an open hardware-agnostic interchangeable open-source format for storing and optimizing machine learning models. The format supports different neural networks architectures as well as some of classical models. What is great about the ONNX is that there is a special runtime for ONNX models which can be used for an inference in different scenarios. 
  
There are a few ways to convert a transformer model to the ONNX format. A high-level one is `optimum-cli` tool which is an addon to the transformers library. In is meant to convert transformer models to different formats including ONNX and optimize them for a fast inference. The full list of architectures supported by the optimum can be [found here](https://huggingface.co/docs/optimum/exporters/onnx/overview).  
There is also a low-level way to convert a transformer model to the ONNX format, you can [dive deeper in here](https://pytorch.org/docs/stable/onnx.html).  
For an additional quality-check during converting via optimum you have to install the `accelerate` package.  
  
```  
optimum-cli export onnx --model intfloat/multilingual-e5-large / \  
 --task feature-extraction \ --library-name transformers \ --framework pt \ converted/
 ```  
  
Optimum-CLI gets only one required arguments which are model name or the path to a model. Optionally you can add a task flag from the predefined list. If this parameter is not set optimum tries to automatically infer it from a model. Also we will set library name and original framework of the model.   
You can see the full list of accepted parameters [here](https://huggingface.co/docs/optimum/en/exporters/onnx/usage_guides/export_a_model#exporting-a-model-to-onnx-using-the-cli).  
  
Let’s check that output of the converted model matches with the output of the original one.  
We’ll use absolute and relative error to evaluate numerical stability after conversion.  
  
## Connecting parts of the pipeline  
  
Triton can’t automatically create instances of preprocessing and postprocessing   
so you need to implement them yourself.   
Fortunately, it is quite easy.   
I use the same method as for the baseline deployment.  
  
Now we have all parts of our pipeline deployed.   
Do we have to manually call them one by one on client?   
No! Triton has an answer for that. Even two answers, actually.   
[Model Ensemble](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/architecture.html#ensemble-models)   
and [BLS (Business Logic Scripting](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/python_backend/README.html#business-logic-scripting).   
Ensemble is purposed for pipelines that can be represented as directed acyclic graphs   
i.e. those which do not have loops and conditional flows.   
It is enough for our pipeline.   
  
  
## TensorRT Acceleration  
  
It is well known that models with weights in fp32 can be accelerated   
through conversion to fp32 or quantization to int8.   
ONNX Runtime contains TensorRT Execution Provider which can accelerate ONNX model   
in runtime including conversion to fp16.   
While conversion to fp16 can give you noticeable boost   
in inference performance you should always check that   
converted model gives close outputs to an original one   
e.g. the intfloat/multilingual-e5-large model   
based on XLMRoberta architecture contains LayerNorm   
layers which known to cause problems when converted to fp16.   
So preferably you should use mixed precision mode with such models   
converting to fp16 all layers except LayerNorm.   
  
A basic configuration of an tensorrt acceleration in a model’s proto-config looks like this:  
```prototext  
optimization { execution_accelerators {  
 gpu_execution_accelerator : [ { name : "tensorrt" parameters { key: "precision_mode" value: "FP16" } parameters { key: "trt_layer_norm_fp32_fallback" value: "true"} parameters { key: "max_workspace_size_bytes" value: "4294967296" } }]}}  
```  
  
According to [Triton ONNX Runtime docs](https://github.com/triton-inference-server/onnxruntime_backend):  
* precision_mode: The precision used for optimization. Allowed values are "FP32", "FP16" and "INT8". Default value is "FP32".  
* max_workspace_size_bytes: The maximum GPU memory the model can use temporarily during execution. Default value is 1GB.  
* [trt_layer_norm_fp32_fallback](https://onnxruntime.ai/docs/execution-providers/TensorRT-ExecutionProvider.html#trt_layer_norm_fp32_fallback): force Pow + Reduce ops in layer norm to FP32. Allowed values are “true” and “false”.  
  
  
So how exactly TensorRT Execution Provider does optimize a model during runtime?   
Firstly, it tries to fuse layers as much as it can so the provider   
can replace them with high-performance one optimized for specific hardware used in runtime.   
  
But in general, creating an engine from scratch is an expensive operation. After fusing layers the provider performs timing tests to choose the quickest implementations to the actual GPU in runtime.   
These tests require few inputs so it means that your model will perform very slow in the beginning of a deployment.   
But this warmup can be done before a model will be ready for requests. Triton model definition have a dedicated section for this.  

In general you should experiment with `max_workspace_size_bytes` parameter to find the best performance.
  
```prototext  
model_warmup [  
 { name: "onnx_ort_warmup_min" batch_size: 1 inputs: { key: "attention_mask" value: { data_type: TYPE_INT64 dims: [ 4 ] input_data_file: "1/raw_attention_mask" } } inputs: { key: "input_ids" value: { data_type: TYPE_INT64 dims: [ 4 ] input_data_file: "1/raw_input_ids" } } }]  
```  
  
As our model requires two inputs we define them both in these section with theirs data types,  
dimensions and corresponding data files.  
The script for the data files preparation can be found [here]().  
It is important to notice TensorRT Provider can [rebuild the model on-the-go](https://huggingface.co/docs/optimum/en/onnxruntime/usage_guides/gpu#tensorrt-engine-build-and-warmup)  
if it gets a request containing at least one of the dimensions less or more than corresponding dimension seen before.  
So we can plan a warmup beforehand.  
We are gonna inference our model with maximum batch size of 256  
and we know that maximum number of input tokens for multilingual-e5-large model is 512.  
We will make two sets of warmup files.  
First will be of batch size 1 and contain 4 number of tokens   
i.e. certainly most inputs will contain more tokens.  
Second one will be of batch size 256 and contain 512 tokens for each sample.  
  
So the final configuration for a warmup will be like this:  
  
```prototext  
model_warmup [  
 { name: "onnx_ort_warmup_min" batch_size: 1 inputs: { key: "attention_mask" value: { data_type: TYPE_INT64 dims: [ 4 ] input_data_file: "1/raw_attention_mask" } } inputs: { key: "input_ids" value: { data_type: TYPE_INT64 dims: [ 4 ] input_data_file: "1/raw_input_ids" } } }, { name: "onnx_ort_warmup_max" batch_size: 256 inputs: { key: "attention_mask" value: { data_type: TYPE_INT64 dims: [ 512 ] input_data_file: "256/raw_attention_mask" } } inputs: { key: "input_ids" value: { data_type: TYPE_INT64 dims: [ 512 ] input_data_file: "256/raw_input_ids" } } }]  
```  
  
The embeddings quality stays the same: (paste distribution of differences)  
  
## Performance comparison  
  
GRPC vs REST clients  
  
GRPC has better performance than REST in general.  
But one of the latest Triton Inference Server update notice   
says that there are some unclear reasons why REST client works much slower than GRPC.  
So we conduct an experiment and also compare different protocols performance.  
  
## What's next?  
  
This post obviously can't cover all aspects of deploying a model with Triton Inference Server.   
So I'd like to tell you what you can try next if the result doesn't satisfy you or you need better performance.  
  
Firstly, to boost performance you can use `instance_groups` section of model configuration  
to launch more than one instance of model. You can read more about it [here](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/model_configuration.html#instance-groups).  
It can add performance to some model or lower performance depending on a model.   
Therefore it is up to you to experiment with your model and find the best conditions.  
  
It is also worth mentioning that there is some problem with combination  
of versions of CUDA driver and ONNX Runtime which can lead  
to failure on the start of the model with `instance_groups[].count`  
more than one if TensorRT acceleration is enabled.  
  
Secondly, you can quantize your model to int8 or even two-bit format.   
To quantize to int8 you can use [Olive] which is a recommended instrument to optimize ONNX   
models for a specific hardware.  
  
There is also an option to convert a model to TensorRT beforehand and inference it with Triton  
using [TensorRT backend](https://github.com/triton-inference-server/tensorrt_backend).  
  
To tune a model to inference on Triton Inference Server you can use [model analyzer](https://github.com/triton-inference-server/model_analyzer) and [performance analyzer]. This is the tool  
provided by Triton to optimize an arbitrary model.  
  
[Polygraphy](https://github.com/NVIDIA/TensorRT/tree/main/tools/Polygraphy) is another tool that is especially useful to analyze a model if it behaves differently on TensorRT.   
It allows to compare inputs and outputs of ONNX model and corresponding TensorRT model layer by layer.   
  
And last but not least, if you optimized a model as well as you can and you still need better performance,  
there is a way to speed up preprocessing by writing your own custom RUST backend. This can give   
a boost cause an underlying implementation of transformers tokenizers is written in RUST  
and during an execution in python the runtime get some overhead from transferring data between  
python interpreter and RUST.   
Postprocessing can be optimized by batching and moving calculations to GPU   
or also can be rewritten in RUST.   
The high-level requirements to implement a custom Triton backend can be found [here](https://docs.nvidia.com/deeplearning/triton-inference-server/archives/triton_inference_server_220/user-guide/docs/backend.html#backend-shared-library).  
  
We in Wildberries use helm charts and Kubernetes for deploying Triton.  
Is allows us to use rolling updates and automatic horizontal autoscaling.  
If you need more features e.g. built-in support of canary deployments and A/B testing you should consider  
[Seldon Core](https://github.com/SeldonIO/seldon-core).  
Seldon Core allows to deploy almost any type of machine learning models.  
Triton is just one type of the many runtimes supported by Seldon Core.  
  
## Resources  
  
- [Triton Inference Server](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/index.html)  
- [Triton Backend for the ONNX Runtime](https://github.com/triton-inference-server/onnxruntime_backend)  
- [Triton Backend for TensorRT](https://github.com/triton-inference-server/tensorrt_backend)  
- [Optimum: Transformers Extension](https://huggingface.co/docs/optimum/en/index)  
- [Polygraphy: A Deep Learning Inference Prototyping and Debugging Toolkit](https://github.com/NVIDIA/TensorRT/tree/main/tools/Polygraphy)  
- [Olive: Hardware-aware Model Optimization Tool](https://github.com/microsoft/Olive)  
- [Seldon Core: Tool to deploy ML Models n Kubernetes at Scale.](https://github.com/SeldonIO/seldon-core)  
- [Locust: Load Testing Tool](https://locust.io/)
