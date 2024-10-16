+++
title = "Deploying a Sentence Transformer with Triton Inference Server"
date = "2024-11-01"
comment = false
draft = false

[taxonomies]
tags=["triton", "inference", "transformers", "onnxruntime", "tensorrt"]

[extra]
comment = false
+++


## Introduction  
  
I was recently assigned to update a fine-tuned retrieval model running on the Triton Inference Server.
The model was underperforming, and the customer required lower latency and a higher requests-per-second (RPS) rate.
The initial deployment used pure PyTorch, so I needed to convert the model to ONNX and apply some optimisations.
It was an interesting journey, so I thought I'd share my experience.

You can ask a question. Why do we use a special inference server instead of just packing a model to FastAPI Service?
Well, there are quite a few reasons. First and foremost, FastAPI isn't developed to be a solution for deploying machine learning models.
FastAPI is best at what is was meant for: high-load backend services with easy-to-hard business logic. Secondly,
FastAPI doesn't provide resources control such as GPU utilisation which is essential to ML models performance. 
Last but not least, if you need advanced optimizations such as dynamic batching, caching and i/o-binding
you need to write them from scratch. Triton Inference Server is designed to be efficient where FastAPI isn't meant to and 
to provide a fine-grained control of all of your computing power in a most effective way.
  
## Overview  
  
In this tutorial, I will walk you through deploying the `intfloat/multilingual-e5-large` model using the Triton Inference Server.
We'll also explore how to convert the model to the ONNX format and optimise it with the TensorRT Execution Provider for ONNX Runtime.
The code is executed on the following server configuration: <SERVER_CONFIG>.
All the code is available in [this repository](https://github.com/rproskuryakov/triton-sentence-transformer-tutorial).
So, let's get started!
  
## Benchmarking Methodology  
  
Therefore, I will use [locust.io](https://locust.io/) for this tutorial.  
  
## Baseline: Python Backend  

Triton Inference Server provides multiple backends for variety of models from boosting models to LLMs.

The simplest way to deploy a model with Triton Inference Server is to package it with Python backend
and perform computations via [Hugging Face pipeline](https://huggingface.co/docs/transformers/v4.44.0/en/main_classes/pipelines).

Python backend requires the model implementation to be placed in the <model_name>/1/model.py file.
The standard directory structure for deploying a model on Triton can be [found here](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/model_repository.html#repository-layout).

First, you must define a class named `TritonPythonModel` and implement the `execute` method.
Optionally, you can also implement the `initialize` and `finalize` methods 
if your model requires loading resources before execution or performing any clean-up before exiting.


```python
from transformers import pipeline  
import triton_python_backend_utils as pb_utils  
  
  
class TritonPythonModel:  
    """
    Your Python model must use the same class name.
    Every Python model that is created must have "TritonPythonModel" 
    as the class name. 
    """  
    def initialize(self, args):  
        self._pipeline = pipeline(  
            "feature-extraction",
            model="intfloat/multilingual-e5-large",
        )  
        
    def execute(self, requests):  
        responses = []  
        for request in requests: 
            input_ = pb_utils.get_input_tensor_by_name(request, "text")  
            input_string = input_.as_numpy()  
            requests_texts = [i[0].decode("utf-8") for i in input_string]  
  
            batch = self.tokenizer(  
                requests_texts,
                max_length=512,
                padding=True,
                truncation=True,
                return_tensors="np",
            ) 
            responses.append(  
                pb_utils.InferenceResponse(  
                    output_tensors=[
                        pb_utils.Tensor(  
                            "output_input_ids",
                            batch["input_ids"],
                        ), pb_utils.Tensor(  
                            "output_attention_mask",
                            batch["attention_mask"],
                        )
                    ]
                )
            )  
        return responses  
```  
  
In the model configuration file, you must define a model name and choose a backend.  
  
```prototext
name: "multilingual-e5-large"  
backend: "python"  
```  
  
You also need to define inputs and outputs of the model which include names, data types and dimensions.  
The string type always has a size of one dimension.  
  
```prototext
input [  
 { 
   name: "text"
   data_type: TYPE_STRING
   dims: [ 1 ]
 }
]  
output [  
 { 
   name: "output_vector"
   data_type: TYPE_FP32
   dims: [ 1024 ]
 }
]  
```  
  
  
## What is ONNX  
  
ONNX is an open, hardware-agnostic, interchangeable open-source format for storing and optimising machine learning models.
It supports various neural network architectures as well as some classical models.
One of the advantages of ONNX is the availability of a dedicated runtime for ONNX models,
which can be used for inference in different scenarios.

There are several methods for converting a transformer model to ONNX format.
A high-level approach is to use the `optimum-cli` tool,
which is an extension of Transformers library.
This tool is designed to convert transformer models to various formats,
including ONNX, and to optimise them for fast inference.
The complete list of architectures Optimum supports can be [found here](https://huggingface.co/docs/optimum/exporters/onnx/overview).

Alternatively, there is a low-level method for converting a transformer model to ONNX.
You can [explore this in more detail here](https://pytorch.org/docs/stable/onnx.html).
For an additional quality check during conversion with Optimum,
you will need to install `accelerate` package.
 
```bash  
optimum-cli export onnx --model intfloat/multilingual-e5-large  \  
 --task feature-extraction  --library-name transformers  --framework pt  converted/
 ```  
  
Optimum-CLI requires only one argument: the model name or the path to the model. Optionally, you can include a task flag from a predefined list. If this parameter is not provided, Optimum will attempt to infer it automatically from the model. We will also specify the library name and the original framework of the model.  
You can view the complete list of accepted parameters [here](https://huggingface.co/docs/optimum/en/exporters/onnx/usage_guides/export_a_model#exporting-a-model-to-onnx-using-the-cli).  

Let’s verify that the output of the converted model matches the output of the original model.  
We’ll use absolute and relative error to assess numerical stability following the conversion.
  
## Connecting parts of the pipeline  
  

Triton cannot automatically create instances of preprocessing and postprocessing, so you will need to implement them yourself. Fortunately, this is quite straightforward.
I use the same method as for the baseline deployment.

Now that all parts of our pipeline are deployed, do we need to call them manually on the client individually?
Not at all! Triton provides solutions for this.
There are two options: Model Ensemble and BLS (Business Logic Scripting).
Ensemble is designed for pipelines that can be represented as directed acyclic graphs,
meaning those without loops and conditional flows. This is sufficient for our pipeline.
  
  
## TensorRT Acceleration  
  
It is well known that models with weights in FP32 can be accelerated by converting them to FP16.
ONNX Runtime includes TensorRT Execution Provider, which can enhance the performance of an ONNX model at runtime,
including conversion to FP16. While converting to FP16 can provide a noticeable boost in inference performance,
it is crucial to verify that the converted model produces results close to those of the original model.
For instance, the `intfloat/multilingual-e5-large` model, based on XLMRoberta architecture,
contains LayerNorm layers known to present issues when converted to FP16.
Therefore, it is preferable to use mixed precision mode with such models,
converting all layers to FP16 except for LayerNorm. 
  
A basic configuration of TensorRT acceleration in a model’s proto-config looks like this:  
```prototext
optimization {
  execution_accelerators { 
    gpu_execution_accelerator : [
      { 
        name : "tensorrt"
        parameters { key: "precision_mode" value: "FP16" }
        parameters { key: "trt_layer_norm_fp32_fallback" value: "true"}
        parameters { key: "max_workspace_size_bytes" value: "4294967296" }
      }
    ]
  }
}  
```  
  
According to [Triton ONNX Runtime Backend docs](https://github.com/triton-inference-server/onnxruntime_backend):  
* precision_mode: The precision used for optimization. Allowed values are "FP32", "FP16" and "INT8". The default value is "FP32".  
* max_workspace_size_bytes: The maximum GPU memory the model can use temporarily during execution. The default value is 1GB.  
* [trt_layer_norm_fp32_fallback](https://onnxruntime.ai/docs/execution-providers/TensorRT-ExecutionProvider.html#trt_layer_norm_fp32_fallback): force Pow + Reduce ops in layer norm to FP32. Allowed values are “true” and “false”.  
  
  
So, how exactly does TensorRT Execution Provider optimise a model during runtime? 

Firstly, it attempts to fuse layers wherever possible so that the provider can replace them with high-performance ones
that are optimised for the specific hardware in use at runtime.

However, creating an engine from scratch is generally an expensive operation.
After fusing layers, the provider conducts timing tests to select the fastest implementations for the actual GPU.
These tests require a few inputs, meaning your model may perform quite slowly at the beginning of deployment.
Fortunately, this warm-up phase can be carried out before the model is ready to handle requests.
The Triton model definition includes a dedicated section for this purpose.

You should generally experiment with the `max_workspace_size_bytes` parameter to achieve the best performance.
  
```prototext
model_warmup [  
 { 
   name: "onnx_ort_warmup_min"
   batch_size: 1
   inputs: {
     key: "attention_mask"
     value: { 
       data_type: TYPE_INT64
       dims: [ 4 ]
       input_data_file: "1/raw_attention_mask"
     } 
   } 
   inputs: {
     key: "input_ids"
     value: {
       data_type: TYPE_INT64
       dims: [ 4 ]
       input_data_file: "1/raw_input_ids"
     } 
   } 
 }
]  
```  


As our model requires two inputs, we define both in this section, specifying their data types, dimensions, and corresponding data files.  
The script for preparing the data files can be found here (INSERT LINK).  
It is important to note that TensorRT Provider can [rebuild the model on the go](https://huggingface.co/docs/optimum/en/onnxruntime/usage_guides/gpu#tensorrt-engine-build-and-warmup)  
if it receives a request with at least one dimension differing from previously encountered.  
Therefore, we should plan for a warm-up phase.  
We will infer our model with a maximum batch size of 256, and we are aware that the maximum number of input tokens for the multilingual-e5-large model is 512.  
We will create two sets of warm-up files. The first will have a batch size of 1 and contain 4 tokens, as most inputs are expected to include more tokens.  
The second will have a batch size 256 and contain 512 tokens per sample.
 
So the final configuration for a warmup will be like this:  
  
```prototext
model_warmup [
  { 
    name: "onnx_ort_warmup_min"
    batch_size: 1
    inputs: { 
      key: "attention_mask"
      value: { 
        data_type: TYPE_INT64
        dims: [ 4 ]
        input_data_file: "1/raw_attention_mask" 
      } 
    }
    inputs: { 
      key: "input_ids"
      value: { 
        data_type: TYPE_INT64
        dims: [ 4 ]
        input_data_file: "1/raw_input_ids" 
      } 
    } 
  }, 
  { 
    name: "onnx_ort_warmup_max" 
    batch_size: 256 
    inputs: { 
      key: "attention_mask"
      value: { 
        data_type: TYPE_INT64
        dims: [ 512 ]
        input_data_file: "256/raw_attention_mask"
      } 
    } 
    inputs: { 
      key: "input_ids" 
      value: { 
        data_type: TYPE_INT64
        dims: [ 512 ] 
        input_data_file: "256/raw_input_ids" 
      } 
    } 
  }
]  
```  
  
The embedding quality stays the same: (paste distribution of differences)  
  
## Performance comparison  
  
GRPC vs REST clients  
  
GRPC performs better than REST in general.  
However, one of the latest Triton Inference Server update notices   
says there are some unclear reasons why the REST client works much slower than GRPC.  
So, we experimented and also compared the performance of different protocols.  
  
## What's next?  
  
This post, naturally, cannot cover every aspect of deploying a model with Triton Inference Server.
So, I would like to suggest some steps you can take if the results do not meet your expectations or if you require improved performance.

Firstly, to enhance performance, you can utilise the `instance_groups` section of the model configuration to launch multiple instances of a model.
Further details can be found [here](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/model_configuration.html#instance-groups).
This approach may improve performance for some models, though it might have the opposite effect depending on the model.
It is therefore advisable to experiment with your model to determine the optimal configuration.

It is also worth noting that there may be compatibility issues between certain versions of CUDA driver and ONNX Runtime,
which can cause failures when starting the model with `instance_groups[].count` set to more than one, especially if TensorRT acceleration is enabled.

Secondly, you can quantise your model to int8, int4 or even a two-bit format.
For quantisation to int8, Olive (INSERT-LINK) is a recommended tool for optimising ONNX models for specific hardware.

Alternatively, you can convert a model to TensorRT beforehand and use Triton for inference via [Triton TensorRT backend](https://github.com/triton-inference-server/tensorrt_backend).

To tune Triton parameters for a model to infer on Triton Inference Server, you can employ the [model analyser](https://github.com/triton-inference-server/model_analyzer) and performance analyser (INSERT-LINK).
These tools, provided by Triton, help in optimising any model.

[Polygraphy](https://github.com/NVIDIA/TensorRT/tree/main/tools/Polygraphy) is another useful tool for analysing a model,
particularly if it behaves differently on TensorRT.
It enables comparison of inputs and outputs of an ONNX model with the corresponding TensorRT model layer by layer.

Finally, if you have optimised your model as much as possible and still require better performance,
consider speeding up preprocessing by developing a custom backend e.g. written in Rust.
This can provide a performance boost, as the underlying implementation of transformer tokenisers is written in Rust.
During execution in Python, some overhead is associated with transferring data between Python interpreter and Rust.
Postprocessing can also be enhanced through batching and offloading calculations to a GPU, or by rewriting it in Rust.
High-level requirements for implementing a custom Triton backend can be found [here](https://docs.nvidia.com/deeplearning/triton-inference-server/archives/triton_inference_server_220/user-guide/docs/backend.html#backend-shared-library).

At Wildberries, we use Helm charts and Kubernetes to deploy models.
This setup allows us to perform rolling updates and automatic horizontal autoscaling.
If you need additional features, such as built-in support for canary deployments and A/B testing,
you might consider [Seldon Core](https://github.com/SeldonIO/seldon-core).
Seldon Core supports the deployment of nearly any type of machine learning model,
the Triton is just one of the many runtimes it supports.

You can also try KServer which has a great feature
to scale deployments to zero if there are not any traffic incoming.

## Bonus: production considerations

opentelemetry, prometheus, seldon-core
  
## Resources  
  
- [Triton Inference Server](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/index.html)  
- [Triton Backend for the ONNX Runtime](https://github.com/triton-inference-server/onnxruntime_backend)  
- [Triton Backend for TensorRT](https://github.com/triton-inference-server/tensorrt_backend)  
- [Optimum: Transformers Extension](https://huggingface.co/docs/optimum/en/index)  
- [Polygraphy: A Deep Learning Inference Prototyping and Debugging Toolkit](https://github.com/NVIDIA/TensorRT/tree/main/tools/Polygraphy)  
- [Olive: Hardware-aware Model Optimization Tool](https://github.com/microsoft/Olive)  
- [Seldon Core: Tool to deploy ML Models with Kubernetes at Scale.](https://github.com/SeldonIO/seldon-core)  
- [Locust: Load Testing Tool](https://locust.io/)