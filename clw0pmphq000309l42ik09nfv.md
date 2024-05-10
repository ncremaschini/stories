---
title: "Evaluating Performance: A Benchmark Study of Serverless Solutions for Message Delivery to Containers on AWS Cloud - Episode 2"
datePublished: Fri May 10 2024 13:25:30 GMT+0000 (Coordinated Universal Time)
cuid: clw0pmphq000309l42ik09nfv
slug: evaluating-performance-a-benchmark-study-of-serverless-solutions-for-message-delivery-to-containers-on-aws-cloud-episode-2
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/ObweQkF5w30/upload/93d10777cdfea4f77c17ec6c38c9b33b.jpeg
tags: aws, dynamodb, serverless, ecs, aws-fargate, aws-eventbridge, fan-out

---

This post follows [my previous post on this topic](https://haveyoutriedrestarting.com/evaluating-performance-a-benchmark-study-of-serverless-solutions-for-message-delivery-to-containers-on-aws-cloud), and it measures the performance of another solution for the same problem, **how to forward events to private containers using serverless services and fan-out patterns.**

## Context

Suppose you have a cluster of containers and you need to notify them when a database record is inserted or changed, and these changes apply to the internal state of the application. A fairly common use case.

Let's say you have the following requirements:

* The tasks are in an autoscaling group, so their number may change over time.
    
* A task is only healthy if it can be updated when the status changes. In other words, all tasks must have the same status. Containers that do not change their status must be marked as unhealthy and replaced.
    
* When a new task is started, it must be in the last known status.
    
* Status changes must be in near real- time. Status changes in the database must be passed on to the containers in less than 2 seconds.
    

## Solutions

In the [first post about this](https://haveyoutriedrestarting.com/evaluating-performance-a-benchmark-study-of-serverless-solutions-for-message-delivery-to-containers-on-aws-cloud) i explored two options and measured performance of this one:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715345902732/dc354ddc-2aba-48a5-9f0c-ec197aad66bf.png align="center")

1. The AppSync API receives mutations and stores derived data in the DynamoDB table
    
2. The DynamoDB streams the events
    
3. The Lambda function is triggered by the DynamoDB stream
    
4. The Lambda function sends the events to the SNS topic
    
5. The SNS topic sends the events to the SQS queues
    
6. The Fargate service reads the events from the SQS queues
    
7. If events are not processed within a timeout, they are moved to the DLQ
    
8. A Cloudwatch alarm is triggered if the DLQ is not empty
    

### The even more serverless version

An even more serverless version of the above solution replaces Lambda and SNS with Eventbridge

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715345130561/1850fc36-77b4-4725-8829-af76b7fff366.png align="center")

1. The AppSync API receives mutations and stores derived data in the DynamoDB table
    
2. The DynamoDB stream the events
    
3. EventBridge is used to filter, transform and...
    
4. ...fan-outs events to SQS queues
    
5. The Fargate service reads the events from the SQS queues
    
6. If events are not processed within a timeout, they are moved to the DLQ
    
7. A Cloudwatch alarm is triggered if the DLQ is not empty
    

*The only code i wrote here is the code to consume SQS from my application, no glue-code is required.*

### Key system parameters

* Region: eu-south-1
    
* Number of tasks: 20
    
* Event bus: 1 SQS per task, 1 DLQ per SQS, all SQS subscribed to one SNS
    
* SQS Consumer: provided by AWS SDK, configured for long polling (20s)
    
* Task configuration: 256 CPU, 512 Memory, Docker image based on [**Official Node Image 20-slim**](https://hub.docker.com/layers/library/node/20-slim/images/sha256-80c3e9753fed11eee3021b96497ba95fe15e5a1dfc16aaf5bc66025f369e00dd?context=explore)
    
* DynamoDB Configured in PayPerUseMode, stream enabled
    
* EventBridge configured to intercept and forwards all events from Dynamo stream to SQS queues
    

### Benchmark parameters

I used a basic postman collection runner to perform a mutation to Appsync every 5 seconds, for 720 iterations.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715346159540/388312f1-2fff-4da0-a809-b1ec05e2d26b.png align="center")

### Goal

The goal was to verify if containers would be updated within 2 seconds, and to verify performance against [the first version](https://haveyoutriedrestarting.com/evaluating-performance-a-benchmark-study-of-serverless-solutions-for-message-delivery-to-containers-on-aws-cloud).

### Measurements

i used the following Cloudwatch provided metrics:

* Appsync latency
    
* Dynamo stream latency
    
* EventBridge Pipe duration
    
* EventBridge Rules latency
    

The SQS time taken custom metric is calculated from SQS provided attributes.

### Results

Here screenshots from my Cloudwatch dashboard

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715346550027/8998f0ec-c31a-494e-8e13-3ef113042c08.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1715346567264/4ffedbdc-6908-4ffe-9838-4739465ab5cf.png align="center")

Few key data, from Average numbers:

* Most of the time is taken by EventBridge rule, I couldn't do anything to lower this latency. The rule is as simple as possible and it is integrated natively by AWS.
    
* The average total time taken is **210.74 ms**, versus **108.39 ms** taken by the first version with Lambda and SNS.
    
* The average response time measured by my client, which covers my client's network latency, is 175 ms. Given Appsync AVG Latency is 62.7 ms, my Avg network latency is 112,13 ms. This means that from my client sending the mutation to consumers receiving the message there are 175 + 113.13 = **288.13 ms**
    

## Conclusion

This solution has proven to be fast and reliable and requires little configuration to set up and no glue-code to write.

Since everything is managed, there is no space for tuning and improvements.

The latency of this solution is worse than the first version by **194.44%.**

However, EventBridge offers many more capabilities than SNS.

## Wrap up

In this article, I have presented you with a solution that I had to design as part of my work and my approach to solution development: this includes clarifying the scope and context, evaluating different options, and having a good knowledge of the parts involved and the performance and quality attributes of the overall system, writing code and benchmarking where necessary, but always with the clear awareness that there are no perfect solutions.

I hope it was helpful to you, and [here is the GitHub repo to deploy both versions of the solution](https://github.com/ncremaschini/fargate-notifications).

Bye 👋!