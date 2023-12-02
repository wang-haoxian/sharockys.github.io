---
author: "Haoxian WANG"
title: "[MLOps] Data Management in MLOps"
date: 2023-11-30T11:00:06+09:00
description: "Discussion on data management in MLOps based on my personal experiences."
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
I have checked several ways to manage data in MLOps, and I will share my experience with you. 

## Why do we need to manage data in MLOps?
There are saying that most of works of data scientists are data cleaning and data preprocessing.   
This is true.
In the machine learning lifecycle, data is the most important part. The quality of data will directly affect the performance of the model. Garbage in, garbage out. 
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
  - Silver: Processed data
  - Gold: Business data
