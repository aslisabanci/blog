---
title: "Building an end-to-end machine learning pipeline as a microservice"
date: "2020-05-30T22:12:03.284Z"
slug: "end-to-end-machinelearning-pipeline-microservice"
template: "post"
draft: false
category: "Coursera"
tags:
  - "aws"
  - "data-engineering"
  - "machine-learning"
description: "In one of my projects, my work focused on building end-to-end machine learning pipelines for training and inference phases of a machine learning model. Head over to this post for diving into those, component by component."
---

In one of my projects, my work focused on building end-to-end machine learning pipelines for training and inference phases of a machine learning model. While I was working on building the pipeline, the model was iteratively developed and optimized by my data scientist team mates in parallel. The project in general consisted of loosely coupled microservices. Our pipeline would be interacting with these services through messaging services of the AWS platform.

During the model training phase, the pipeline would be used to preprocess and structure the data as per the taste buds of our machine learning model. Then, during the inference phase, the pipeline would get the input data; apply the same preprocessing and structuring steps; feed this input into the model; get its inference and make it available for other components’ digestion on this ecosystem.

Let’s go over these pipelines, component by component:

### Data Collection and Enrichment
Our core proprietary data that was collected from different internal sources. Because of this, it needed standardization, normalization and some cleanup with respect to our domain experts’ knowledge. Along with these tasks, enriching this core with 3rd party providers’ data was one of the main and ongoing tasks of this project - as the core data kept getting updated and our enrichment ideas kept continuing.

The scope of this component was:

- Evaluating 3rd party data providers with respect to their technical API specs
- Collecting data from the chosen providers through API requests or through custom processes like FTP deliveries
- Versioning and managing the updates on this continuously growing ground truth data

### Data Storage
Different components of our pipeline dealt with different types of data. Some of them were highly relational while some were highly unstructured and document-style. So, I used both relational and non-relational database systems for storage.

Designing the relational data structure was an ongoing process and my main objective was to keep the data “simply-enough” structured; thus easy to maintain. While evaluating on which database system to use and how best to store the data, I kept discussing questions like these:

* What types of queries do we want to perform on this data?
* How often are we going to want to do this?
* What additions or updates can we foresee on this data?
* What’s our scaling needs?

### Quantitative and Qualitative Checks on Data
We had certain criteria on our minds, when evaluating whether to go with a 3rd party provider’s data or not. A few examples of these were as follows.
#### Quantitative Criteria
* How much of our data cannot be enriched by this provider’s data?
* Can the API perform as per our performance requirements?

#### Qualitative Criteria
* When compared to similar data providers’ data, how similar/different/reliable is this data?
* How good is their technical support? How responsive is the provider?


### Standardization, normalization and Aggregation of Data
As mentioned in [Data Collection and Enrichment](#Data Collection and Enrichment), our data was like a sink where the input was flowing through many different internal and external faucets. So, this multi-regional sink was one of the biggest beasts to tackle throughout our project.

#### Standardization
We had different non-standardized representations of some data; that in fact represent the same entity. To determine our standardization algorithms and processes; I first worked together with our domain experts. After gathering some “business rules” from their side, I worked out the technical process of this standardization and made this step a part of our pipeline going forward.

#### Normalization
This should come as no surprise that the numerical features in our data were in many different ranges. Also, different providers would use different units for the data that we want to consolidate. So, normalization was an important part of our data preprocessing.

#### Aggregation
For certain data, storing every single data point meant that we would require lots of storage and we indeed do not need that much granularity. For these cases, calculating meaningful mathematical aggregations and storing them was a better choice.

### Deployment of the pipeline
Our pipeline had a Docker image, so that it could be registered on ECR and run in an isolated fashion on our microservice ecosystem; while allowing for easy scaling.

### Getting input at inference time
The containerized applications were listening to a specific queue on AWS SQS to get their input. Our machine learning model was able to digest batch input. Thus, for efficiency, the input messages on SQS contained bundled requests. It was the pipeline’s job to unbundle these; do the preprocessing steps separately for each input; and when all the processing is done, bundle the data into a ready-to-be-consumed format and feed this as a batch into our model.

### Outputting the predictions
Our model’s predictions were to be consumed by other microservices on our ecosystem. Just like the input, this communication was also carried over AWS SQS messaging platform.

As a final note; this project opened my eyes to the reality of building machine learning applications. I experienced that 90% of our project was about getting the training data right. Because of this, most of the effort is rightfully spent on this matter to get a high performing model in the end.

Afterwards, when it’s time to put this application on production, the performance of the pipeline gets the spotlight. If our end-to-end pipeline could not complete the steps of enriching & preprocessing the input data and getting the model’s inference in a timely manner, this would be a total showstopper. As Miles Davis puts it: *"Time isn't the main thing. It's the only thing."*
