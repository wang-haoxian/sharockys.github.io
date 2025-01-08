---
author: "Haoxian WANG"
title: "[ML] Huggingface Pipeline as a Service"
date: 2023-04-15T12:00:06+09:00
description: "Quick setup of Huggingface Pipeline as a Service Using Python Web Servers"
draft: false
hideToc: false
enableToc: true
enableTocContent: false
author: Haoxian
authorEmoji: 👻
tags: 
- Transformers
- Huggingface
- NLP
- ML
- Python
- Web Server
- Starlette
- FastAPI
- PyTorch
- MLOps
---

## Huggingface Pipeline as a Service 
I am currently working on some projects using Huggingface Transformers. I found it is a little bit tedious to set up a web server for each model. So I decided to write a script to automate the process.  
There is a great documentation on Huggingface website about how to set up a web server for a model using Scarlette. Here is the link: [pipeline_webserver](https://huggingface.co/docs/transformers/main/en/pipeline_webserver)  

## Quick Setup with Huggingface's script in documentation
Here is the code that I tweaked a little bit for my use case.  You can find it in my github repo: [hf_serve](https://github.com/haoxian-lab/hf-serve)    

For the devices, you can always use `cpu` or `cuda` for GPU.   
And if you are working on Mac OS with Apple Silicon, you can use `mps` to accelerate the inference. More about PyTorch with MPS: [PyTorch with MPS](https://developer.apple.com/metal/pytorch/). You will need to install the experimental version of PyTorch with MPS support.  
With MPS, you can get a 2x to 3x speedup on inference. 


In the repo, the default model used is  [cardiffnlp/tweet-topic-21-multi](https://huggingface.co/cardiffnlp/tweet-topic-21-multi) .   
You can change it in the `hf_serve/config.py` file.  

```python   
# hf_serve/serving_starlette.py
from starlette.applications import Starlette
from starlette.responses import JSONResponse
from starlette.routing import Route
from transformers import pipeline
import asyncio
from hf_serve.config import DEVICE, MODEL 


async def homepage(request):
    payload = await request.body()
    string = payload.decode("utf-8")

    response_q = asyncio.Queue()
    await request.app.model_queue.put((string, response_q))
    output = await response_q.get()
    return JSONResponse(output)


async def server_loop(q):
    pipe = pipeline(model=MODEL, top_k=None, device=DEVICE)
    while True:
        (string, response_q) = await q.get()
        out = pipe(string)
        await response_q.put(out)


app = Starlette(
    routes=[
        Route("/", homepage, methods=["POST"]),
    ],
)


@app.on_event("startup")
async def startup_event():
    q = asyncio.Queue()
    app.model_queue = q
    asyncio.create_task(server_loop(q))
```

Then you can execute it with 
```bash
uvicorn hf_serve.serving_fastapi:app --host 0.0.0.0 --port 8000
```

To test it, you can use curl or postman to send a POST request to the server.  
```bash
curl -X POST -H "Content-Type: application/json" -d '{"text": "Hello world"}' http://localhost:8000
```

You will see something like this: 
```json
[
  [
    {
      "label": "diaries_&_daily_life",
      "score": 0.6784588694572449
    },
    {
      "label": "other_hobbies",
      "score": 0.16906404495239258
    },
    {
      "label": "relationships",
      "score": 0.1307423859834671
    },
    {
      "label": "film_tv_&_video",
      "score": 0.045063380151987076
    },
    {
      "label": "arts_&_culture",
      "score": 0.04253656417131424
    },
    {
      "label": "celebrity_&_pop_culture",
      "score": 0.02954648993909359
    },
    {
      "label": "music",
      "score": 0.017763793468475342
    },
    {
      "label": "family",
      "score": 0.013196156360208988
    },
    {
      "label": "news_&_social_concern",
      "score": 0.009492534212768078
    },
    {
      "label": "learning_&_educational",
      "score": 0.007527184206992388
    },
    {
      "label": "gaming",
      "score": 0.005994295235723257
    },
    {
      "label": "travel_&_adventure",
      "score": 0.005782891530543566
    },
    {
      "label": "sports",
      "score": 0.005135056562721729
    },
    {
      "label": "business_&_entrepreneurs",
      "score": 0.004528014920651913
    },
    {
      "label": "youth_&_student_life",
      "score": 0.004223798401653767
    },
    {
      "label": "science_&_technology",
      "score": 0.0038613160140812397
    },
    {
      "label": "food_&_dining",
      "score": 0.0025304360315203667
    },
    {
      "label": "fashion_&_style",
      "score": 0.0022351129446178675
    },
    {
      "label": "fitness_&_health",
      "score": 0.0014648066135123372
    }
  ]
]
```

In the original documentation, the model prediction only shows the best label. But in my case, I want to see all the labels and their scores. So I changed the code a little bit.   


## Use FastAPI as Web Server
I also tried to use FastAPI as web server. This is only because I am more familiar with FastAPI and there is no particular taste for me.  
And this makes it a good exercice to learn Starlette and FastApi.

The code is in the `hf_serve/serving_fastapi.py` file.  
```python
# hf_serve/serving_fastapi.py
from fastapi import FastAPI
from fastapi import Request, Response
from transformers import pipeline
import asyncio
from hf_serve.config import DEVICE, MODEL


app = FastAPI()


@app.post("/")
async def homepage(request: Request, response: Response):
    payload = await request.body()
    string = payload.decode("utf-8")

    response_q = asyncio.Queue()
    await app.model_queue.put((string, response_q))
    output = await response_q.get()
    response.status_code = 200
    return output


@app.on_event("startup")
async def startup_event():
    q = asyncio.Queue()
    app.model_queue = q
    asyncio.create_task(server_loop(q))


async def server_loop(q):
    pipe = pipeline(model=MODEL, top_k=None, device=DEVICE)
    while True:
        (string, response_q) = await q.get()
        out = pipe(string)
        await response_q.put(out)
``` 

The test will be the same as for Starlette.  

## Dockerize the Web Server 
To dockerize the web server, I created a `Dockerfile`. 
```dockerfile
# Use the official Python image as the base image
FROM nvidia/cuda:11.7.1-cudnn8-runtime-ubuntu22.04

# Set the working directory in the container
WORKDIR /app

# Copy the requirements file into the container
COPY pyproject.toml .
COPY poetry.lock .
COPY README.md .
COPY hf_serve ./hf_serve
# Install the necessary packages
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        libgomp1 \
        python3.10 \
        python3-pip \
        && \
    rm -rf /var/lib/apt/lists/*

RUN ln -s /usr/bin/python3 /usr/bin/python 

RUN pip install poetry && poetry config virtualenvs.create false && poetry install && rm -rf ~/.cache

# Install pytorch with cuda
RUN pip install torch  --index-url https://download.pytorch.org/whl/cu118 --no-cache-dir



# Expose the port that the application will run on
EXPOSE 80

# Start the application when the container starts
CMD ["python", "-m", "uvicorn", "hf_serve.serving_fastapi:app", "--host", "0.0.0.0", "--port", "80"]
```

It takes the `nvidia/cuda:11.7.1-cudnn8-runtime-ubuntu22.04` image as the base image.  
Then it installs the necessary packages and pytorch with cuda.  
Finally, it starts the web server. 

To build the image, you can use the following command. 
```bash
docker build -t hf_serve .
```

To run the image, you can use the following command. 
```bash
docker run -p 8000:80 hf_serve
```

If you want to use the GPU, you can use the following command. 
```bash
docker run --runtime=nvidia -p 8000:80 hf_serve
```

## Conclusion & Performance Test
In this article, I showed how to use the `transformers` library to serve a HF Pipeline with Starlette and FastAPI.    
FastAPI is not always the best option. It depends on the use case. Since it's widely used, it's a good option to learn.  
If you are curious about the performance, you can check the [benchmark](https://web-frameworks-benchmark.netlify.app/result?l=python).  
In this benchmark, Starlette is way faster than FastAPI with almost 40% of difference. (Later on, I cheked on the FastAPI's doc and find out that FastAPI is fully compatible with (and based on) Starlette.)
It would be also interesting to see how the performance changes with the number of requests.In the next article, I will use [Locust](https://github.com/locustio/locust) to benchmark the performance of the web server with the pipeline. 
I want to see also how torchserve can improve the performance compared to these two web servers.  
Then let's see how LRU cache/ Redis can improve the performance of server load in a next article.
