---
title: "Building an automated workflow through Bitbucket and AWS components"
date: "2020-05-30T22:12:03.284Z"
slug: "automated-workflow-bitbucket-aws"
template: "post"
draft: false
category: "Automated Workflow"
tags:
  - "aws"
  - "data-engineering"
  - "bitbucket"
  - "batch"
  - "lambda"
description: "In one of my projects, I built an automated workflow so that whenever an entity on Bitbucket gets updated through a push to a certain repository, some of the files in that repo would be transformed into different files by a serverless application on AWS."
---

In one of my projects, I built an automated workflow so that whenever an entity on Bitbucket gets updated through a push to a certain repository, some of the files in that repo would be transformed into different files by a serverless application on AWS. This transformation creates some intermediary files which need to be processed by a containerized app. Finally, the output of this app would be stored again on AWS and the owner of the Bitbucket push request would be notified by email with a link to reach this output.


Let’s briefly go over the connected parts and mention the AWS components I used to create this skeleton.

## Automated workflow

So, here’s what happens step-by-step:

1. Somebody makes a push on a Bitbucket repository.
2. My Lambda function, sitting behind AWS API Gateway gets triggered by this push. This happens via the webhook that I set up on this Bitbucket repository.
3. The Lambda function (i.e. the aforementioned “serverless application on AWS”) checks out this repository from Bitbucket as it needs to access the files in there. After getting those files, it performs certain transformations on certain files and creates some artifacts.
4. When the Lambda function successfully creates these intermediary files, it sends a message to SQS.
5. This SQS message triggers a batch job on AWS Batch.
6. The batch job is set up to fire up a container using a specified Docker image on ECR, to start the aforementioned “containerized app”.
7. This containerized app takes the intermediary files as its input, does its job on them and when its successfully finished, notifies the owner of the Bitbucket push in step number 1.

## CI/CD workflow for my own deployment automation
During the development of this skeleton, I also developed a CI/CD workflow for my own deployment automation. Whenever I made changes on my code, I wanted two things to happen:

* My Lambda function gets updated and re-deployed
* My Docker image gets re-pushed to ECR
so that the Lambda function and the “containerized app” of this workflow reflect the latest changes on my app code.

My code repository was also sitting on Bitbucket. So, what I did was:

1. Whenever I make a push to my repository, my whole repository gets copied to a folder on AWS S3.  
2. I set up AWS CodeBuild to automatically build this app folder.
3. Then I connected CodeBuild with AWS CodeDeploy to re-deploy my Lambda functions and the built Docker image on ECR.
Doing these two deployments manually is a huge hassle after every code change and not-doing this automation would be a very inefficient use of my time. So building this CI/CD for myself was absolutely a good investment.



Sounds fun, right? Well, at least it was for me. This integration surely gave me many headaches but these are the type of headaches that I quite enjoy. When I solve those headaches and finally see a component getting integrated with another one, I love that feeling of having solved a problem and I guess that’s a big part of why I love my profession.
