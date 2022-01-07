---
title: "Serving a machine learning model’s predictions through AWS Lambda and API Gateway"
date: "2020-30-05T22:12:03.284Z"
slug: "serving-predictions-aws-lambda-gateway"
template: "post"
draft: false
category: "Coursera"
tags:
  - "aws"
  - "data-engineering"
  - "machine-learning"
  - "lambda"
  - "dynamodb"
  - "api-gateway"
description: "In one of my projects, I worked on persisting the periodic output of an already developed machine learning model after enriching it with some metadata and then serving this enriched output through an API on AWS API Gateway."
---

In one of my projects, I worked on persisting the periodic output of an already developed machine learning model after enriching it with some metadata and then serving this enriched output through an API on AWS API Gateway.

Let’s go into the details component by component:

## Persistence and enrichment of the model output
Our machine learning model was persisting its periodic output as a file on S3. The consumers of our model’s output needed not only the model’s predictions but also certain metadata along with it; so that they don’t need to perform any extra queries on their side; because speed was of essence as usual.

In order to create a highly responsive API with minimum latency, we needed to make the served data ready ahead in time. To make the model output queryable as per the requirements, the application I built would

1. Read the output file on S3 and process it record by record
2. Join it with some metadata retrieved from AWS Athena
3. Store the consolidated records on AWS DynamoDB in a “ready to be served” structure

## RESTful API on AWS API Gateway
Well, we only needed a GET endpoint for this purpose, so we had nothing special on this layer. We also deferred the request authentication to the Lambda function itself (for business reasons) so the API Gateway did not require much work to setup.

## Request authentication
We made use of Bearer tokens in the request headers to verify the incoming requests’ authentication. The Bearer tokens were JWT tokens and they would contain some information like which public key was used for their signature, when the token expires, etc.. We had these public keys stored on our systems so that we can first verify that an incoming token is indeed a valid token and it has not yet expired. After verifying the token, the Lambda function would decode this token, retrieve certain fields of the decoded structure and use these to query DynamoDB.

## Serving Lambda function
After successfully verifying the requester’s authentication information, and fetching the related records from DynamoDB, our Lambda function would simply structure this data as per our requirements and return it to the caller.

## Data management
We made use of the TimeToLive feature of DynamoDB to get rid of relatively old data that we don’t need for our purposes.

## Service discovery
Our API would be discovered by other microservices and for service discovery, we needed to have certain artifacts on certain locations. The artifacts gave our API consumers some information like which endpoints we’re exposing and at which addresses they can access our API on our dev / test / qa / pre-prod / prod environments.

## Monitoring dashboards
We meticulously monitored our latency and error metrics on CloudWatch and had alarms for certain cases to have our eyes on our API’s health and performance. We also created custom dashboards for monitoring how often some use-cases happen, as we could detect these by filtering our logs against certain log messages.

## API documentation using OpenAPI 3.0 Specs
![api-docs](/media/swagger/swagger_editor.png)
I used The OpenAPI 3.0 specs for the technical documentation of our API, as it has become the de facto standard to specify what an API can do. [Swagger Editor](http://editor.swagger.io/) is a great tool to do this, since it can dynamically validate your specs as you write through a beautiful GUI. If you haven’t adopted conforming to the OpenAPI specs for your APIs, go learn about it now - I’m sure you’ll love this standardization idea!


For me, this project was so fun to work on. It had a few performance optimization challenges, while preparing the ready-to-be-served data and serving the output through our Lambda function. The satisfaction of overcoming those hurdles were directly proportional to the size of those challenges though!
