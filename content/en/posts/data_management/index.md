---
author: "Haoxian WANG"
title: "[MLOps] Data Management in MLOps"
date: 2023-11-30T11:00:06+09:00
description: "My two cents on data management in MLOps based on my personal experiences."
draft: false
hideToc: false
enableToc: true
enableTocContent: false
author: Haoxian
authorEmoji: 👻
tags: 
- MLOps
- Data Management
- NLP
- DVC 
- S3
- MinIO
- MLFlow
---

## What is Data Management in MLOps?
Data management in MLOps is the process of managing data in the machine learning lifecycle. It includes data collection, data storage, data versioning, data quality, data monitoring, data lineage, data governance, and data security.   
In this article, we will focus on data versioning, data storage, and data quality. 
I have checked several ways to manage data in MLOps, and I will share my experience with you. Please feel free to leave a comment if you have any questions or suggestions. This is not a data engineering article, so my focus is not on the data engineering part. I will try to keep it simple and easy to understand.

## Why do we need to manage data in MLOps?
There are saying that most of works of data scientists are data cleaning and data preprocessing.   
This is true.
In the machine learning lifecycle, data is the most important part. The quality of data will directly affect the performance of the model. As we what we say all the time: "Garbage in, garbage out."
Therefore, we need to manage data in MLOps.   
It's a big challenge to manage data in MLOps, because data is always changing. Imagine that you have a dataset collected from production, and you have trained a model with the dataset. After a while, the data in production has changed. You need to retrain the model with the new dataset. How can you manage the data and the model? For a simple model, you may have many different kind of processings. Take HTML as an example, you may have different processings like:
  - removing tags
  - removing stop words
  - removing punctuation
  - removing numbers
  - removing special characters
  - Normalizing URLs
  - removing emojis 
  - removing non-target langugae words 
  - removing non-ASCII characters 
  - removing non-printable characters   

For some of the processings, you can do it on the fly while training/inference, but some of them you need to do it before training since it's not your part of pipeline. Or you will need to align with other downstream users of the same data source. For the sake of cost, you may want to do it once and for all but with possible different versions. 
Meanwhile, the pipeline is separated to several steps, and each step may have different requirements for the data. For example, the first step may need the raw data, and the second step may need the data after removing tags.
When the dataset is big enough, it's hard to manage the data. Thus we need to manage the data in MLOps. 

## What are the requirements for data management in MLOps?
I am not here to talk about using very advanced tools like Delta Lake or Apache Iceberg. They have some fancy features like time travel. But for a small team or a small project, they are too heavy and the cost may be too high. So I will focus on the principles of data management in MLOps by taking some ideas from data engineering. 
From my personal experiences, the requirements for data management in MLOps are:
  - Traceability: We need to know where the data comes from, and how the data is processed. The metadata of the data should be managed.
  - Reproducibility: We need to be able to reproduce the data with certain procedures in case we delete the data temporarily.
  - Versioning: Data should be managed with versions, and we need to be able to access the data with different versions. 
  - Scalability: The ability to manage the data in a scalable way with business growth.
  - Flexibility: The possibility of changing the processing of the data. 
  - Cost: The cost should be reasonable for the business.
  - Performance: There are data requires cocurrent read/write, and we need to manage the data with performance.
  - Simplicity: New members should be able to understand the data management and tools easily.
  - Automation: The pipeline should be able to run automatically with as less as possible human intervention, just like a CI/CD pipeline.
  - Collaboration: The management process should allow multiple people to work on the same data. 
  - Security: We need to define the scope of access control for different people.

This is not an exhaustive list, but it's a good start. 
I separate these requirements into three groups: 
  - Traceability, Reproducibility, Versioning
  - Scalability, Flexibility, Cost, Performance
  - Simplicity, Automation, Collaboration, Security

### Traceability, Reproducibility, Versioning 
These three requirements represents the most general ideas in people's mind when talking about data management. I put them together because they are highly coupled. 
Traceability means we need to know where the data comes from, and how the data is processed. 
Reproducibility means we need to be able to reproduce the data with some given steps. 
Versioning means we need to be able to manage the versions of the data incrementally. 
These three requirements together means we are able to understand at each step when and how the data is processed, with what kind of upstream data source, and then we are able to reproduce each step with the information we have.
What we need for these three requirements are basically a set of informations to describe the runs of the pipeline.

### Scalability, Flexibility, Cost, Performance
These four elements together force us to think about the most efficient way to manage the data. Sevceral questions we need to ask ourselves are:  

- How to manage the data when the data is big enough?
- How to manage the data with the changing business requirements?
- How to manage the data with a reasonable cost?    
  
We need to find out the solution to make a balance between these four elements. It's easy to use a huge Google BigQuery to manage the data at a very scalable and performant way, at the price of potentially sacrificing the flexibility and it may be too expensive. It's also easy to allocate unlimited size of bucket storage in AWS S3, but it may be too expensive and we may lose some performance. Some advanced modern data warehouses like Snowflake may be a good choice, but it add too much overhead for a small team or a small project.   
All I want to say it's that there are no silver bullet, and we need to find out the best solution for our own case. Sometimes git-lfs is our best friend, sometimes it's not. Sometimes a simple S3 bucket will be nice. And in many cases DVC could be cool at some point.   

### Simplicity, Automation, Collaboration, Security
These four elements comes together when there is more than one person working on the same project.  
If the data is difficult to access, it will be hard to collaborate. 
Collaboration comes with security concerns.   
And bad automation will make things more complicated.
Automation could be the source of secret leaks. 
A good solution should be at the balance of these four elements.
There are so many tools that can help with us. For example, Argo Workflow, Airflow, Dagster, etc. 
Basically, we need a centralized place to click and run the pipeline, and we need to be able to manage the access control without knowing every detail of the pipeline. 

## Data Design Pattern
### The Medallion Architecture 
There are many ways to manage data in MLOps. I will introduce the Medallion Architecture for it's simplicity and flexibility. From this super cool article [The Medallion Architecture](https://www.databricks.com/glossary/medallion-architecture) from Databricks, we can quickly build an idea of how to manage data in MLOps. 
There are basically three layers in the Medallion Architecture: 
  - Bronze: Raw data
  - Silver: Cleaned and conformed data
  - Gold: curated business-level data / Feature Store

#### Bronze Layer
The Bronze layer is the raw data layer. It's the data that is collected from the data source. It has not been processed yet, it may contain some garbage data.Let's say there are data in format like JSON, but it's not well structured. You may see some missing fields, or some fields are not in the right format. It's not impossible that there are so many repeated data. You normally cannot read it directly without some processing.
Generally it's so called unstructured or semi-structured data. It's the data that is closest to the data source. 

#### Silver Layer
The Silver layer is the cleaned and conformed data layer. It's the data that has been processed and cleaned. It may not contain all the information for the final tasks. For example, if you want to build a dataset to fine-tune a LLM for insurance domain, this layer may contain the data that is anonymized. It may not contain the data that is not related to the insurance domain.

#### Gold Layer
The Gold layer is the curated business-level data layer. It's the data that is ready to use for the final tasks. A good example is the Feature Store. It's the data that is ready to use for the training and inference. It's the data that is ready to use for the final tasks. If you are building a Retrieval Augmented Generation (RAG) system, it could be the Vector DB that contains the vectors of the documents. 


#### Free to choose the data store
With the Medallion Architecture, we can manage the data flexibly with necessary control without stuck to a single data storage. For example, the raw data can be stored in a S3 bucket with versioning, and the cleansed and bussiness data can be stored in a database like Postgres or Delta Lake. Each layer can be managed with different tools with different requirements, or if your organization has a centralized data platform, you can use the same tool to manage all the layers. It's important to adopt the right tool for the right layer based on your business requirements.

## Data Quality
Data quality is much more than just data cleaning. It's the process of ensuring the data is fit for the purpose. The data should be accurate, complete, consistent, and timely. However, in practice, it's not easy to achieve all the requirements. In MlOps, we tend to begin with relatively low quality data, and then improve the quality of the data incrementally. For example, the classic human-in-the-loop approach is a good way to improve the quality of the data and align with the business requirements. 
There are also tools that can help us to improve the quality of the data. For example, [Great Expectations](https://greatexpectations.io/gx-cloud) is a good tool to help us to define the expectations of the data, and then validate the data with the expectations. It's a good way to improve the quality of the data. There are also other tools like [Apache Griffin](https://griffin.apache.org), etc. But these are a little bit too heavy for a small team or a small project. 
Another kind of data quality tools is something like [CleanLab](https://github.com/cleanlab/cleanlab). It's a tool to help us to identify the mislabeled data by analyzing the outliars. The strong connection between data quality and model quality makes it a good tool to improve the quality of the data semi-automatically.

## Data Versioning
### General tools for data versioning
Data versioning is the process of managing the versions of the data. It's important to keep track of the versions of the data, and be able to access the data with different versions, hence we can reproduce the data with the same version. 
There are many tools that can help us to manage the versions of the data. For example, [DVC](https://dvc.org/), [git-lfs](https://git-lfs.com/), [Neptune](https://docs.neptune.ai/tutorials/data_versioning/), [Pachyderm](https://www.pachyderm.com/), [lakeFS](https://github.com/treeverse/lakeFS), etc. 

While DVC and git-lfs are more like a git extension, Neptune, Pachyderm, and lakeFS are more like a data lake. They are more like a centralized data platform. For a restrained budget, DVC and git-lfs are more suitable. For a big organization, tools like Neptune, Pachyderm, and lakeFS are good choices. And sometimes the infra team may be reluctant to add another tool to the stack, so we may need to use the existing tools like git-lfs.

### Home made solution
Sometimes we don't need to use any tools but well defined procedures. For example, we can use a S3 bucket to store the data, and use the folder structure to manage the versions of the data. It's simple and easy to understand, it's scalable with the business growth, and it's cost effective. The only drawback is that it's not easy to manage the access control in some fine-grained way. But it's not a big problem for a small team or a small project. With a proper defined toolkit and workflow, it's easy to manage the data with versioning this way. 

## Describe the data with metadata
The dataset is often not obvious to understand. For instance, say you use parquets to store data, and you have a parquet file with 100 columns. It's not easy to understand the data without any metadata. If you use a data warehouse like Snowflake or Delta Lake, you have easy access to the metadata of the data because they are built-in. But if you use a S3 bucket to store the data, you need to manage the metadata by yourself.   
In the point of view of data engineering, the major metadata of the data is the schema of the data. In the point of view of data science, the major metadata of the data is the statistics of the data. From a business point of view, the metadata we want to see is a self-explanatory description of the data which matches the business logic. And in NLP, we may want to see the text normalization rules, the tokenization rules, etc. This is to say, the metadata is a complex set of information that varies from different perspectives and use cases. For me, we need to manage the metadata in a strucutred way, and we need to be able to access the metadata easily.   

### Accessibility of the metadata 
The metadata should be positionned alongside the dataset. For databases, they are built-in. For more generic file systems like S3, we need to manage the metadata by ourselves. I would recommand formats that are easy to read and write, like JSON, YAML, or TOML, etc. This allows us to preview the metadata easily.   
The reason to choose this kind of formats, is that we may want to use the metadata in the pipeline. For example, we may want to use the metadata to validate the data, or we may want to use the metadata to generate the documentation of the data. This requires the data to be in a structured format which can be easily parsed.   
Sometimes an extract of the data is a good way to describe the data. It's a good idea to put an sample of the data alongside the dataset. We may not want to download the whole dataset to build a first impression of the data. And we can use the sample to explain the whole dataset and the pipeline to non technical people.

### What to add to the metadata
I have made a list of possible metadata that we may want to manage:
  - Description: What is the data about?
  - Data source: What's the upstream data source?
  - SHA1: Useful to check data integrity
  - Schema: Allows us to load the data easily
  - Statistics: help us to handle the data (like missing values, etc.)
    - Number of rows
    - Number of columns
  - Sample: A sample of the data if the data is compatible with the format, otherwise you may want it to be in a separate file
  - ETL pipeline stage: What's the current stage of the data in the pipeline?
  - Exectuion time of the pipeline 
  - Mapping between the columns and the business logic

Personally, there is always a rule of thumb to determine the complexity of something. That is, if someone new onboarding the project, how long will it take for him/her to understand the thing. The metadata here should be auto-representative not only for the data itself, but also for the pipeline or the code. 


## Example of HTML data for NLP with the Medallion Architecture
We have talked about HTML data in the beginning of this article. Let's take a look at how we can manage the data with the Medallion Architecture. Let's assume that you want to build a system to categorize the HTML documents. The data source is a crawler that crawls the HTML documents from the Internet, at a rate of 1000 documents per day. The output of the crawler is a JSON file that contains the HTML documents stored in a S3 bucket. 
In this example, we will limit the tools to show the case in a restrained budget. So we only use git and S3 with MLFlow. All workflows run on ArgoWorkflow or Airflow.
Let's name this project as `html-classification` and let's begin.
### Pipeline
The pipeline will be essentially something like this:
![Data Pipeline Illustration](<pipeline overview.png>)

### Architecture 
The architecture is illustrated as below:
![Data Architecture Illustration](<data management overview.png>)

### Overview of different layers
#### The nomclature for the paths to the data in S3
We are going to setup a bunch of rules since we are using S3 to store the data. We will use the following nomclature for the paths to the data in S3:
```yaml
s3://<bucket_name>/html-classification/<stage>/<date-of-data>/<dataset>
```

#### Bronze Layer
At this stage, the most direct input is the HTML documents collected by the crawler. It can be the output of an ingestion workflow that streams the documents from some message queue system to process it in a batch way. The output of the ingestion workflow is a JSON file that contains the HTML documents. As a personal preference, I use JSONL format to facilitate the reading in streaming mode.  
The format looks like: 
```Json
# s3://<bucket_name>/html-classification/raw/2021-09-01/html-documents.json
{"url": "https://www.example.com/giberish_html_?","html": "<html><body>...</body></html>","timestamp": "2021-09-01T00:00:00+09:00"}
{"url": "https://www.example2.co/giberish_html_?","html": "<html><body>...</body></html>","timestamp": "2021-09-02T00:00:00+09:00"}
...
```
Metadata:
```yaml
-
Description: The raw HTML documents collected by the crawler. 
Data source: The crawler
Date of data creation: 2021-09-01
workflow-name: crawler-to-s3
SHA1: 8b8ae027744d2f920eb1aeee676d589957de40cc
Schema: 
  - url: string
  - html: string
  - timestamp: string
Statistics:
  - Number of rows: 1000
  - Number of columns: 3
Sample:
  - url: "https://www.example.com/giberish_html_?"
    html: "<html><body>...</body></html>"
    timestamp: "2021-09-01T00:00:00+09:00"
  - url: "https://www.example2.co/giberish_html_?"
    html: "<html><body>...</body></html>"
    timestamp: "2021-09-02T00:00:00+09:00"
```

#### Silver Layer
At this stage, we begin to have some much more structured data.

##### URL Normalization
The first step is to normalize the URLs, with some simple deduplication with title (This is a rather arbitrary process as example only. In real life, the dedup should be much more complex and taking account more information. For example, the same page with different timestamps. We tend to keep the fresh copy). The output of this step will be like:
```Json
# s3://<bucket_name>/html-classification/preprocess/url-normalization/2021-09-02/html-documents.json
{"url": "https://www.example.com","html": "<html><body>...</body></html>","timestamp": "2021-09-01T00:00:00+09:00"}
{"url": "https://www.example2.com","html": "<html><body>...</body></html>","timestamp": "2021-09-02T00:00:00+09:00"}
...
```
Metadata:
```yaml
-
Description: The raw HTML documents collected by the crawler, with basic deduplication and normalization of the URLs.
Data source: s3://<bucket_name>/html-classification/raw/2021-09-01/html-documents.json
Date of data creation: 2021-09-02
workflow-name: ingestion
SHA1: da4eec8e1ffe93df6b8a768ac78d98b3879a5baa
Schema: 
  - url: string
  - html: string
  - timestamp: string
Statistics:
  - Number of rows: 998 # 2 duplicated rows removed
  - Number of columns: 3
Sample:
  - url: "https://www.example.com"
    html: "<html><body>...</body></html>"
    timestamp: "2021-09-01T00:00:00+09:00"
  - url: "https://www.example2.com"
    html: "<html><body>...</body></html>"
    timestamp: "2021-09-02T00:00:00+09:00"
```
This will allow us to have unique URLs for the same document, and the url can be used as the primary key of the data.

##### HTML Parsing
This time the format will be like:
```Json
# s3://<bucket_name>/html-classification/preprocess/html-parsing/2021-09-02/html-documents.json
{"url": "https://www.example.com","title": "Example","content": "This is an example.","timestamp": "2021-09-01T00:00:00+09:00"}
{"url": "https://www.example2.com","title": "Example2","content": "This is an example2.","timestamp": "2021-09-02T00:00:00+09:00"}
...
```
Metadata:
```yaml
-
Description: HTML documents with title and content extracted. 
Data source: s3://<bucket_name>/html-classification/preprocess/url-normalization/2021-09-02/html-documents.json
Date of data creation: 2021-09-02
workflow-name: html-parsing
SHA1: 8b8ae027744d2f920eb1aeee676d589957de40cc
Schema: 
  - url: string
  - title: string
  - content: string
  - timestamp: string
Statistics:
  - Number of rows: 998 
  - Number of columns: 4
Sample:
  - url: "https://www.example.com"
    title: "Example"
    content: "This is an example."
    timestamp: "2021-09-01T00:00:00+09:00"
  - url: "https://www.example2.com"
    title: "Example2"
    content: "This is an example2."
    timestamp: "2021-09-02T00:00:00+09:00"
```

This is just an illustration format. To be more realistic, we may want to have more information like the metadata of the page, language of the document, the encoding of the document, etc. But hey, we can use some NLP tools to process the data. 

##### Text Normalization
Now we want to apply some text normalization rules to the content. The choice of the rules depends on the business requirements. For example, we may want to remove the stop words, remove the punctuation, remove the numbers, remove the special characters, etc. And this could make huge difference for the final performance of the model. In this example, we could have several different versions of the data with different text normalization rules. And the metadata should come in handy to help us to understand the data.
###### Version 1
```Json
# s3://<bucket_name>/html-classification/preprocess/text-normalization/v1/2021-09-02/html-documents.json
{"url": "https://www.example.com","title": "example","content": "this is an example.","timestamp": "2021-09-01T00:00:00+09:00"}
{"url": "https://www.example2.com","title": "example2","content": "this is an example2.","timestamp": "2021-09-02T00:00:00+09:00"}
...
```
Metadata:
```yaml
-
Description: HTML documents with title and content extracted, and text normalization applied. 
Data source: s3://<bucket_name>/html-classification/preprocess/html-parsing/2021-09-02/html-documents.json
Date of data creation: 2021-09-03
workflow-name: text-normalization-v1
process: 
  - remove punctuation
  - remove special characters
SHA1: 8b8ae027744d2f920eb1aeee676d589957de40cc
Schema: 
  - url: string
  - title: string
  - content: string
  - timestamp: string
Statistics:
  - Number of rows: 998 
  - Number of columns: 4
Sample:
  - url: "https://www.example.com"
    title: "example"
    content: "this is an example"
    timestamp: "2021-09-01T00:00:00+09:00"
  - url: "https://www.example2.com"
    title: "example2"
    content: "this is an example2"
    timestamp: "2021-09-02T00:00:00+09:00"
```
###### Version 2
```Json
# s3://<bucket_name>/html-classification/preprocess/text-normalization/v2/2021-09-02/html-documents.json
{"url": "https://www.example.com","title": "example","content": "this be an example.","timestamp": "2021-09-01T00:00:00+09:00"}
{"url": "https://www.example2.com","title": "example2","content": "this be an example2.","timestamp": "2021-09-02T00:00:00+09:00"}
...
```
Metadata:
```yaml
-
Description: HTML documents with title and content extracted, and text normalization applied.
Data source: s3://<bucket_name>/html-classification/preprocess/html-parsing/2021-09-02/html-documents.json
Date of data creation: 2021-09-03
workflow-name: text-normalization-v2
process: 
  - remove special characters
  - verbe lematization
SHA1: 8b8ae027744d2f920eb1aeee676d589957de40cc
Schema: 
  - url: string
  - title: string
  - content: string
  - timestamp: string
Statistics:
  - Number of rows: 998 
  - Number of columns: 4
Sample:
  - url: "https://www.example.com"
    title: "example"
    content: "this be an example."
    timestamp: "2021-09-01T00:00:00+09:00"
  - url: "https://www.example2.com"
    title: "example2"
    content: "this be an example2."
    timestamp: "2021-09-02T00:00:00+09:00"
```

Now we have two versions of the data with different text normalization rules. We can use the metadata to understand the data. And we should be able to run model training with different versions of the data.

#### Gold Layer
Now we need to have the data ready for our final task: document categorization. We need to have the data in a format that is ready to use for the training and inference.  
Now we have the basic text input, but we don't have the targets for supervised learning. Let's say we use an external paid API to get the targets. The output of the API is a JSON file that contains the targets. 
```Json
# s3://<bucket_name>/html-classification/annotated/2021-09-04/html-documents.json
{"url": "https://www.example.com", "text": "example - this be an example.", "category": "entertainment", "timestamp": "2021-09-01T00:00:00+09:00"}
{"url": "https://www.example2.com", "text": "example2 - this be an example2.", "category": "politics", "timestamp": "2021-09-02T00:00:00+09:00"}
...
```
Metadata:
```yaml
-
Description: HTML documents with title and content extracted, and text normalization applied, and targets added.
Data source: 
  - s3://<bucket_name>/html-classification/preprocess/text-normalization/v2/2021-09-03/html-documents.json # This allows us to trace the previous steps
  - https://www.example.com/api/v1/document-categorization  # the api 
Date of data creation: 2021-09-04
workflow-name: annotation
SHA1: 8b8ae027744d2f920eb1aeee676d589957de40cc
Schema: 
  - url: string
  - title: string
  - content: string
  - category: string
  - timestamp: string
Statistics:
  - Number of rows: 998 
  - Number of columns: 5
Sample:
  - url: "https://www.example.com"
    title: "example"
    content: "this be an example."
    category: "entertainment"
    timestamp: "2021-09-01T00:00:00+09:00"
  - url: "https://www.example2.com"
    title: "example2"
    content: "this be an example2."
    category: "politics"
    timestamp: "2021-09-02T00:00:00+09:00"
``` 
We use the v2 version of the data, and we add the targets to the data. Now we have the data ready for the training and inference.   
When we train the model, we can push the metadata to MLFlow, as artifact or as parameter. This allows us to trace the data and the model.  

#### Post Train Layer
This is the step after the model training. We refine the data in this step with CleanLab. We try to rectify the mislabeled data and remove some outliers that are not related to the business. 
```Json
# s3://<bucket_name>/html-classification/post-train/2021-09-05/html-documents.json
{"url": "https://www.example.com", "text": "example - this be an example.", "category": "entertainment", "timestamp": "2021-09-01T00:00:00+09:00"}
{"url": "https://www.example2.com", "text": "example2 - this be an example2.", "category": "politics", "timestamp": "2021-09-02T00:00:00+09:00"}
```
Metadata:
```yaml
-
Description: HTML documents with title and content extracted, and text normalization applied, and targets added, and mislabeled data removed.
Data source: 
  - s3://<bucket_name>/html-classification/preprocess/annotated/2021-09-04/html-documents.json # This allows us to trace the previous steps
Date of data creation: 2021-09-05
workflow-name: clean-lab
SHA1: 8b8ae027744d2f920eb1aeee676d589957de40cc
Schema: 
  - url: string
  - title: string
  - content: string
  - category: string
  - timestamp: string
Statistics:
  - Number of rows: 900 # reduced a lot of bad data 
  - Number of columns: 5
Sample:
  - url: "https://www.example.com"
    title: "example"
    content: "this be an example."
    category: "entertainment"
    timestamp: "2021-09-01T00:00:00+09:00"
  - url: "https://www.example2.com"
    title: "example2"
    content: "this be an example2."
    category: "politics"
    timestamp: "2021-09-02T00:00:00+09:00"
```

## Conclusion
This is a very simple data management example. But it shows the basic idea of how to manage data in MLOps.  
It's good to start with something simple and easy to understand. And then we can improve the data management incrementally.  
We can begin with some naïve formats like JSONL, and then we can use more advanced formats like Parquet, Delta Table, etc. We can use some simple tools like plain S3FS or git-lfs, and then we can use more advanced tools like Neptune, Pachyderm, lakeFS, etc. We can use some simple workflow tools like Argo Workflow, and then we can use more advanced tools like Airflow, Dagster, etc.   
We can also use MLFlow to track the data we used for the training.    

For me, the most important thing is to be able to track the data the clean way to allow us to acheive the goals we mentioned in the beginning of this article. This example showed we can do it with some simple tools. Minio can help us control the access, the metadata can help us understand the data, and the workflow tools can help us to manage the pipeline. And all this with a restrained budget. 
