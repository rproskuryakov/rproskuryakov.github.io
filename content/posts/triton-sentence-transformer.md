+++
title = "Deploying a Sentence Transformer with Triton Inference Server"
date = "2024-11-01"
draft = true
description = ""
[taxonomies]
tags=["triton", "inference", "transformers", "onnxruntime", "tensorrt"]

[extra]
comment = true
+++


## Introduction  
  
I was recently tasked with updating a fine-tuned retrieval model deployed on the Triton Inference Server.
The model was underperforming, and the customer needed lower latency and a higher requests-per-second (RPS) rate.
Since the deployment was written in pure PyTorch, which isn’t the most efficient solution for production, optimization was necessary.
The process turned out to be quite an interesting journey, so I wanted to share my experience.

You might wonder why we use a specialized inference server instead of simply deploying the model in a FastAPI service. 
There are several reasons for this.

First, FastAPI is not designed specifically for deploying machine learning models;
its strength lies in handling high-load backend services with varying levels of business logic complexity.

Second, FastAPI lacks resource management features like GPU utilization, which is crucial for the performance of ML models.
Additionally, advanced optimizations such as dynamic batching, caching, and I/O binding would need to be implemented manually in FastAPI.
In contrast, Triton Inference Server is built specifically for these tasks, providing efficient use of computational resources and fine-grained control over your infrastructure.
  
## Overview  
  
In this tutorial, I will walk you through deploying the `intfloat/multilingual-e5-large` model using the Triton Inference Server.
We'll also explore how to convert the model to the ONNX format and optimise it with the TensorRT Execution Provider for ONNX Runtime.
The code is executed on the following server configuration: <SERVER_CONFIG>.
All the code is available in [this repository](https://github.com/rproskuryakov/triton-sentence-transformer-tutorial).
So, let's get started!
  
## Benchmarking Methodology  

There are many tools available for load testing, with some of the most well-known being the
[Apache HTTP benchmarking tool (ab)](https://httpd.apache.org/docs/2.4/programs/ab.html),
[JMeter](https://jmeter.apache.org/), and [Locust](https://locust.io/).

While AB is simple to use, it doesn't support gRPC requests,
making it unsuitable for our needs.
JMeter is a strong tool for load testing and does support gRPC requests through a plugin.
However, it requires writing tests in Java, which is too complex for our project.

Therefore, we'll use Locust, which supports gRPC and offers a straightforward Python interface for writing test cases.

## Let's speed up the model

### Baseline: Python Backend  

Triton Inference Server supports multiple backends, enabling the deployment of a wide range of models,
from boosting models to large language models (LLMs).

The easiest way to deploy a model with Triton is to use the Python backend,
allowing computations to be performed via the [Hugging Face pipeline](https://huggingface.co/docs/transformers/v4.44.0/en/main_classes/pipelines).

For the Python backend, the model's implementation needs to be placed in 
the `<model_name>/1/model.py` file.
The standard directory structure for deploying a model on Triton 
can be found in the [official documentation](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/model_repository.html#repository-layout).

To get started, define a class named `TritonPythonModel` and implement the `execute` method.
Optionally, you can also implement the `initialize` and `finalize` methods 
if the model requires resource loading before execution or clean-up tasks before shutting down.


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
  
  
### What is ONNX  
  
ONNX is an open-source, hardware-agnostic format for storing and optimizing machine learning models.
It supports a wide range of neural network architectures as well as some classical models.
One of its key advantages is the availability of a dedicated ONNX runtime,
which can be used for efficient inference across various environments.

There are multiple ways to convert a transformer model to the ONNX format.
A high-level approach involves using the `optimum-cli` tool, an extension of the Transformers library. 
This tool is specifically designed to convert transformer models into various formats,
including ONNX, and optimize them for faster inference.
You can find the complete list of supported architectures [here](https://huggingface.co/docs/optimum/exporters/onnx/overview).

For a more low-level conversion method,
you can refer to the [official PyTorch documentation](https://pytorch.org/docs/stable/onnx.html).
Additionally, if you want to perform a quality check during conversion with Optimum,
you will need to install the `accelerate` package.
 
```bash  
optimum-cli export onnx --model intfloat/multilingual-e5-large \
--task feature-extraction  --library-name transformers  --framework pt  converted/
 ```  
  
Optimum-CLI requires only one argument: the model name or the path to the model. Optionally, you can include a task flag from a predefined list. If this parameter is not provided, Optimum will attempt to infer it automatically from the model. We will also specify the library name and the original framework of the model.  
You can view the complete list of accepted parameters [here](https://huggingface.co/docs/optimum/en/exporters/onnx/usage_guides/export_a_model#exporting-a-model-to-onnx-using-the-cli).  

Let’s verify that the output of the converted model matches the output of the original model.  
We’ll use absolute and relative error to assess numerical stability following the conversion.
  
### Connecting parts of the pipeline  
  

Triton doesn’t automatically handle preprocessing and postprocessing steps,
so you’ll need to implement these yourself.
Thankfully, this is straightforward, and I use the same approach as with the baseline deployment.

Now that our entire pipeline is deployed, do we need to call each part manually from the client?
Not at all – Triton provides solutions for this.
There are two options: Model Ensemble and Business Logic Scripting (BLS).
Model Ensemble is ideal for pipelines that can be represented as directed acyclic graphs (DAGs),
meaning they have no loops or conditional flows, which works perfectly for our pipeline.
  
  
### TensorRT Acceleration  
  
It’s widely known that models using FP32 weights can be accelerated by converting them to FP16.
ONNX Runtime offers a TensorRT Execution Provider,
which improves the performance of ONNX models during runtime,
including converting them to FP16.
While this conversion can significantly boost inference speed,
it's important to ensure that the converted model produces results similar to the original.

For example, the `intfloat/multilingual-e5-large` model, based on the XLMRoberta architecture,
includes LayerNorm layers,
which are known to cause issues when converted to FP16.
As a result, it’s often better to use mixed precision,
converting most layers to FP16 while keeping LayerNorm layers in FP32.
  
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
  
According to the [Triton ONNX Runtime Backend docs](https://github.com/triton-inference-server/onnxruntime_backend):  
* precision_mode: The precision used for optimization. Allowed values are "FP32", "FP16" and "INT8". The default value is "FP32".  
* max_workspace_size_bytes: The maximum GPU memory the model can use temporarily during execution. The default value is 1GB.  
* [trt_layer_norm_fp32_fallback](https://onnxruntime.ai/docs/execution-providers/TensorRT-ExecutionProvider.html#trt_layer_norm_fp32_fallback): force Pow + Reduce ops in layer norm to FP32. Allowed values are “true” and “false”.  
  
  
How does the TensorRT Execution Provider optimize a model during runtime?

First, it attempts to fuse layers wherever possible,
allowing the provider to replace them with high-performance implementations
optimized for the specific hardware used during runtime.

However, building an engine from scratch is a resource-intensive process.
After layer fusion, the provider runs timing tests to determine the fastest implementations for the target GPU.
These tests require several inputs, so the model may initially perform slowly right after deployment.
Fortunately, this warm-up phase can be completed before the model starts handling requests,
and Triton's model definition includes a dedicated section for configuring this.

To achieve optimal performance, it's recommended to experiment with the `max_workspace_size_bytes` parameter.
  
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


Since our model requires two inputs, we need to define both, specifying their data types, dimensions, and corresponding data files.
You can find the script for preparing these data files [here] (INSERT LINK).

It’s important to note that the TensorRT provider can [rebuild the model on the fly](https://huggingface.co/docs/optimum/en/onnxruntime/usage_guides/gpu#tensorrt-engine-build-and-warmup)
if it receives a request with dimensions that differ from those previously encountered.
To account for this, we need to include a warm-up phase.

Our model will infer with a maximum batch size of 256,
and we know the maximum number of input tokens for the `multilingual-e5-large` model is 512.
Therefore, we will prepare two sets of warm-up files: one with a batch size of 1 and 4 tokens,
assuming most inputs will include more tokens, and another with a batch size of 256 and 512 tokens per sample.
 
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
  
Protobuf is a very GRPC performs better than REST in general.  

(INSERT-PLOT-GRPC-vs-HTTP)

(INSERT-PLOT-gRPC-python-onnx-tensorrt)
 
  
## What's next?  
  
This post, of course, cannot cover every aspect of deploying a model with Triton Inference Server.
However, I’d like to suggest some steps you can take if the results don’t meet your expectations
or if you need improved performance.

### Enhancing Performance

First, to boost performance, consider using the `instance_groups` section of the model configuration
to launch multiple instances of your model.
More details on this can be found [here](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/model_configuration.html#instance-groups).
While this approach may enhance performance for some models, it could have the opposite effect for others,
so it's advisable to experiment with your model to find the optimal configuration.

Be aware that compatibility issues may arise between certain versions of the CUDA driver and ONNX Runtime.
These issues can lead to failures when starting the model with `instance_groups[].count` set to more than one,
particularly when TensorRT acceleration is enabled.

### Model Quantization

Another effective strategy is to quantize your model to int8, int4, or even a two-bit format.
For int8 quantization, [Olive](https://github.com/microsoft/Olive) is a recommended tool for optimizing ONNX models
for specific hardware.

Alternatively, you can convert your model to TensorRT ahead-of-time and utilize Triton for inference
through the [Triton TensorRT backend](https://github.com/triton-inference-server/tensorrt_backend).

### Tuning Triton Parameters

To fine-tune Triton parameters for inference, you can use the [model analyzer](https://github.com/triton-inference-server/model_analyzer)
and the [performance analyzer](https://docs.nvidia.com/deeplearning/triton-inference-server/archives/triton-inference-server-2280/user-guide/docs/user_guide/perf_analyzer.html).
These tools provided by Triton assist in optimizing any model.

[Polygraphy](https://github.com/NVIDIA/TensorRT/tree/main/tools/Polygraphy) is another valuable tool
for analyzing model behavior, especially if your model performs differently on TensorRT.
It enables you to compare the inputs and outputs of an ONNX model against the corresponding TensorRT model, layer by layer.

### Custom Backend Development

If you've optimized your model as much as possible and still need better performance, consider speeding up preprocessing by developing a custom backend, perhaps in Rust.
This approach can provide a performance boost since the underlying implementation of transformer tokenizers is in Rust.
When executing in Python, there is some overhead associated with transferring data between the Python interpreter and Rust. 

You can also enhance post-processing through batching and offloading calculations to a GPU or by rewriting the code in Rust.
You can find high-level requirements for implementing a custom Triton backend [here](https://docs.nvidia.com/deeplearning/triton-inference-server/archives/triton_inference_server_220/user-guide/docs/backend.html#backend-shared-library).

### Deployment Strategies

At Wildberries, we deploy models using Helm charts and Kubernetes,
which allows us to perform rolling updates and automatic horizontal autoscaling.
If you’re looking for additional features such as built-in support for canary deployments and A/B testing,
consider [Seldon Core](https://github.com/SeldonIO/seldon-core).
Seldon Core supports the deployment of nearly any type of machine learning model,
with Triton being just one of the many runtimes it accommodates.

You might also explore KServe,
which offers the advantageous feature of scaling deployments down to zero when there is no incoming traffic.

## Bonus: production considerations

[//]: # (In the real production environment reliability and observability are required. )

[//]: # (That includes monitoring and scalability. )

[//]: # (To monitor a model one can use an opentelemetry )

[//]: # (which is an open-source instrument for collect, export and generate metrics, logs an traces of an application.)

[//]: # (Triton [supports]&#40;https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/trace.html&#41; opentelemetry generation for traces of inference requests.)

[//]: # ()
[//]: # (Majority of companies use Prometheus as a database for metrics and Grafana for dashboards and alerts.)

[//]: # (Triton also [supports]&#40;https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/metrics.html&#41; providing prometheus metrics. They contain GPU metrics and requests statistics.)

[//]: # ()
[//]: # ()
[//]: # (Another issue in a production environment is scalability.)

[//]: # ([Seldon-Core]&#40;https://github.com/SeldonIO/seldon-core&#41; is one of the instruments to help with that.)

[//]: # (It is a tool dedicated to deployment of machine learning models on Kubernetes.)

[//]: # (Seldon introduces the concept of a server in fact being a backend for a model.)

[//]: # (Servers include scikit-learn, xgboost etc. Also, they include Triton.)

[//]: # (That means you can run any of the Triton instances via Seldon. )

[//]: # (Seldon out of the box provides scaling, A/B-tests, Canary deployments and many more. )

In a real production environment, reliability and observability are crucial.
This includes effective monitoring and scalability.
To monitor machine learning models, OpenTelemetry, an open-source framework for collecting,
exporting, and generating metrics, logs, and traces, can be used.
Triton [supports](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/trace.html)
OpenTelemetry, enabling the generation of traces for inference requests.

Most companies rely on Prometheus as a metrics database,
along with Grafana for visual dashboards and alerts.
Triton also [supports](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/metrics.html)
Prometheus metrics, including GPU performance data and request statistics, making it easier to monitor and optimize models in production.

Scalability is another key concern in production environments.
Tools like [Seldon-Core](https://github.com/SeldonIO/seldon-core) are designed to address this challenge.
Seldon-Core facilitates the deployment of machine learning models on Kubernetes,
introducing the concept of a "server" that acts as the backend for a model.
It supports popular frameworks like scikit-learn, XGBoost, and Triton.
With Seldon, you can run Triton instances while benefiting from built-in features such as automatic scaling,
A/B testing, Canary deployments, and more.
  
## Resources  
  
- [Triton Inference Server](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/index.html)  
- [Triton Backend for the ONNX Runtime](https://github.com/triton-inference-server/onnxruntime_backend)  
- [Triton Backend for TensorRT](https://github.com/triton-inference-server/tensorrt_backend)  
- [Optimum: Transformers Extension](https://huggingface.co/docs/optimum/en/index)  
- [Polygraphy: A Deep Learning Inference Prototyping and Debugging Toolkit](https://github.com/NVIDIA/TensorRT/tree/main/tools/Polygraphy)  
- [Olive: Hardware-aware Model Optimization Tool](https://github.com/microsoft/Olive)  
- [Seldon Core: Tool to deploy ML Models with Kubernetes at Scale.](https://github.com/SeldonIO/seldon-core)  
- [Locust: Load Testing Tool](https://locust.io/)