---
author: "Haoxian WANG"
title: "[MLOps] Deployment of Whisper on AWS Fargate using Terraform"
date: 2024-02-01T11:00:06+09:00
description: "A quick guide to deploy Whisper on AWS Fargate using Terraform"
draft: false
hideToc: false
enableToc: true
enableTocContent: false
author: Haoxian
authorEmoji: 👻
tags: 
- Whisper
- AWS
- Fargate
- Terraform
- MLOps
- Docker
---

## Introduction  
Whisper is a STT (Speech to Text) model developed by [OPENAI](https://openai.com/research/whisper). It's a powerful model that can convert human speech into text. Friend of mine encoutered this project as an job interview task with IaC using Terraform so I get the idea to do it on my own and I find it interesting to deploy it on AWS Fargate. I chose Fargate because of the highly optimized version of it doesn't require GPUs. In this post, I will share my journey to this final solution and show you how to deploy it.

## Prerequisites 
- AWS CLI configured
- Terraform with Terraform Cloud (or local state if you prefer) configured
- Docker installed

If you don't have any experience on Terraform, you can use the official tutorial to get started: [Getting Started with Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli?in=terraform/aws-get-started) with AWS Provider. 

## Interpretation of the task  
Due to the confidentiality of the task, I can't share the original task. However, I can share the main points of the task. The task is to serve STT model to replace the previous outsourced service. The model is expected to serve 100 users in a call center for post-call analytics and we should expect X10 users in the future.    
My understanding on this task is:   
- This is not about the concurrency of the model. We don't need real-time processing, so we can use batch processing.   
- The number of users is not a significant factor for the service, but the number of audios generated and the processing time for each document. (For example, we want to have the transcriptions of all our last weeks calls to have insights in the weekly meetings this monday, that means the time window to process all the files before monday, so for the files of Friday we will have to process them during the weekend, without taking consideration of the time that analytics team should take). 
- The model is not expected to be used by the public, so we don't need to worry about the security of the model.(At least for this version, we can assume that we only use the model in a secure environment)   
- The cost should be minimized. As long as we can optimize the model to use the least resources, we can use the cheapest service instead of the most powerful one with GPUs.   
- The model should be scalable. We should be able to scale the model to serve more users in the future.  
- We can create S3 buckets to store the audio files and the results, but we don't have to since the end user of the service(the analytics Team) may have already had a solution for this. 
- There are no limit on the choice of the model. We can use any model as long as it can serve the purpose. But we should be able to evaluate the model with some baseline metrics.(e.g. WER, CER, etc. But this is not the main point of the task) 
- The Diarisation is not mentionned but important in the multi-party communications like telephones, but that should be another project since we are talking about another ML module. 

### Reference of solutions on the market 
In France, I have encountered a few companies that provide STT services. I think it's interesting to check out [AlloMedia](https://www.allo-media.net/), a company that provides STT services for call centers. We can be inspired by their solutions.

## Investigation on Whisper and its deployment 
### Frameworks and Libraries 
There are several possible frameworks to use Whisper model, mainly: 

- [OpenAI official](https://github.com/openai/whisper) The official OpenAI implementation 
- [Whisper.CPP](https://github.com/ggerganov/whisper.cpp) The C++ implementation of the model 
- [Transformers](https://huggingface.co/openai/whisper-large-v3) The Huggingface implementation of the model in its transformers library.
- [Faster Whisper](https://github.com/systran/faster-whisper) which converts the model to [CTranslate2](https://github.com/OpenNMT/CTranslate2) format to optimize the model.

There is a benchmark of these implementations on [this page](https://github.com/SYSTRAN/faster-whisper/blob/master/README.md) and I take some of the information here: 

### Large-v2 model on GPU

| Implementation | Precision | Beam size | Time | Max. GPU memory | Max. CPU memory |
| --- | --- | --- | --- | --- | --- |
| openai/whisper | fp16 | 5 | 4m30s | 11325MB | 9439MB |
| faster-whisper | fp16 | 5 | 54s | 4755MB | 3244MB |
| faster-whisper | int8 | 5 | 59s | 3091MB | 3117MB |

*Executed with CUDA 11.7.1 on a NVIDIA Tesla V100S.*

### Small model on CPU

| Implementation | Precision | Beam size | Time | Max. memory |
| --- | --- | --- | --- | --- |
| openai/whisper | fp32 | 5 | 10m31s | 3101MB |
| whisper.cpp | fp32 | 5 | 17m42s | 1581MB |
| whisper.cpp | fp16 | 5 | 12m39s | 873MB |
| faster-whisper | fp32 | 5 | 2m44s | 1675MB |
| faster-whisper | int8 | 5 | 2m04s | 995MB |

*Executed with 8 threads on a Intel(R) Xeon(R) Gold 6226R.*

As we can see, the faster-whisper implementation is the most optimized one. It's interesting to use this implementation for our deployment.

This is why I chose to use CPU and Fargate. After some tests, I found that the base model is very optimized and can be used on CPU and we can get the result in a reasonable time without the cost of quality. 

The code to serve the model is very simple, we can use the following code to serve the model: 

```python
from faster_whisper import WhisperModel

model_size = "large-v3"

# Run on GPU with FP16
model = WhisperModel(model_size, device="cuda", compute_type="float16")

# or run on GPU with INT8
# model = WhisperModel(model_size, device="cuda", compute_type="int8_float16")
# or run on CPU with INT8
# model = WhisperModel(model_size, device="cpu", compute_type="int8")

segments, info = model.transcribe("audio.mp3", beam_size=5)

print("Detected language '%s' with probability %f" % (info.language, info.language_probability))

for segment in segments:
    print("[%.2fs -> %.2fs] %s" % (segment.start, segment.end, segment.text))
```

As you can see, the code is very simple and we can use it to serve the model. 

### Possible Solutions on AWS 
There are several possible way to deploy the model on AWS, mainly:
- EC2: We can deploy the model on EC2 and use the autoscaling group to scale the model.
- Lambda: We can deploy the model on Lambda and use the API Gateway to serve the model.
- Fargate: We can deploy the model on Fargate and use the ECS to serve the model.
- SageMaker: We can deploy the model on SageMaker and use the endpoint to serve the model.
- EKS: We can deploy the model on EKS and use the Kubernetes to serve the model.

I chose to use Fargate because it's the most optimized solution for our case. We don't need GPUs and we don't need to worry about the infrastructure. We can use the Fargate to serve the model and we can use the ECS to scale the model.     

Let's still checkout the pros and cons of each solution:
| Solution | Pros | Cons |
| --- | --- | --- |
| EC2 | Full control of the infrastructure | Need to manage the infrastructure and expensive when using GPU |
| Lambda | Serverless | Limited to 15 minutes and no GPU |
| Fargate | Serverless | No GPU |
| SageMaker | Managed service | Expensive |
| EKS | Full control of the infrastructure | Need to manage the infrastructure |

## Overview of the solution 
I draw the pipeline of the deployment as follows: 
![The illustrated pipeline](whisper-arch-base.png)  

We can expect three parts in the deployment:
- Codes 
  - The Python code for model serving code: The code to serve the model.
  - The Terraform code for infrastructure: The code to deploy the infrastructure.
- The infrastructure part: The infrastructure of code and CI/CD pipeline. This part is rather static and normally in the organization, it's managed by the DevOps team or the Infra team. And it's not the main part of the task and not necessary to be done with AWS. It can be Github Actions, Gitlab CI, Jenkins, etc.
- The model serving part: The model is served by the Fargate and the ECS. We can use the ECS to scale the model.

TO BE CONTINUED
