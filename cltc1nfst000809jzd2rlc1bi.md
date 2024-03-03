---
title: "Evaluating Performance: A Benchmark Study of Serverless Solutions for Message Delivery to Containers on AWS Cloud"
seoTitle: "aws-cloud-serverless-event-delivery"
seoDescription: "A benchmark about serverless message delivery to  container on AWS cloud"
datePublished: Sun Mar 03 2024 21:48:21 GMT+0000 (Coordinated Universal Time)
cuid: cltc1nfst000809jzd2rlc1bi
slug: evaluating-performance-a-benchmark-study-of-serverless-solutions-for-message-delivery-to-containers-on-aws-cloud
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/DX9X0g0Cg88/upload/26f142070bd8ddbbc75f69d594f011a4.jpeg
tags: cloud, aws, performance, dynamodb, benchmark, serverless, sqs, sns, aws-fargate

---

In this article i'll show you how to forward events to private containers using serverless services and fan-out pattern.

I'll explore possible solutions within AWS ecosystem, but all are applicable regardless the actual service / implementation.

## Context

Suppose you have a cluster of containers and you need to notify them when a database record is inserted or changed, and these changes apply to the internal state of the application. A fairly common use case.

Let's say you have the following requirements:

* The tasks are in an autoscaling group, so their number may change over time.
    
* A task is only healthy if it can be updated when the status changes. In other words, all tasks must have the same status. Containers that do not change their status must be marked as unhealthy and replaced.
    
* When a new task is started, it must be in the last known status.
    
* Status changes must be in near real- time. Status changes in the database must be passed on to the containers in less than 2 seconds.
    

Given these requirements, let's explore a few options.

## Option 1: tasks directly querying the database

![Task querying directly the database](https://cdn.hashnode.com/res/hashnode/image/upload/v1708895045088/f09bbb59-736b-4b19-af9d-ea7c163d40b4.jpeg align="center")

### Pros:

* easy to implement: The task is just to perform a simple query and get the current status, assuming it can be queried.
    
* fast: It really depends on the DB resources and the complexity of the query, but there are not many hops and can be configured to be fast. You can configure polling time to match our requirement of 2 seconds requirement, e.g. every 1 second.
    
* easy to mark as unhealthy tasks that fails to perform queries. The application could catch errors in queries and mark itself as unhealthy if it has enough resources. Otherwise, the load balancer's health check would fail.
    

### Cons:

* waste of resources: Your application queries the database even if no changes have been made. If your database does not change more frequently than the polling rate, most queries are useless.
    
* your database is a single point of failure: If the database cannot serve queries, tasks cannot be notified.
    
* it does not scale well: As the number of tasks grows, the number of queries grows and you may need to scale the database as well, or you may need a very large cluster running all the time to accommodate any scaling, wasting resources.
    
* difficult to monitor: How can you check if an individual task is in the right state?
    

In such a scenario, I definitely don't like polling.

Let's try a different and opposite approach.

## Option 2: Db streams changes to containers

Instead of having tasks asking to the database, let's have the database notifying them for changes.

![db pushes events to tasks](https://cdn.hashnode.com/res/hashnode/image/upload/v1708896310371/615cc978-49a9-4efb-9124-0c30d467a5f4.jpeg align="center")

Before go into the pros and cons, i must say that it would be very hard if not impossible to implement this solution exactly as i drown it. We can use a very popular pattern, called *fan-out.*

This is the [wikipedia](https://en.wikipedia.org/wiki/Fan-out_(software)) definition:

> In message-oriented middleware solutions, fan-out is a messaging pattern used to model an information exchange that implies the delivery (or spreading) of a message to one or multiple destinations possibly in parallel, and not halting the process that executes the messaging to wait for any response to that message

To make things a little more concrete, let's use some popular AWS services that are commonly used to implement this pattern:

* DynamoDB: NoSql database with native event streaming
    
* SNS: pub/sub event bus
    
* SQS: queue service
    

The solution looks like this:

![event streaming and fan-out in action](https://cdn.hashnode.com/res/hashnode/image/upload/v1708897206829/cacbad49-053a-40d4-8210-531f54622538.jpeg align="center")

Now let's explore pros and cons:

### Pros:

* first of all, you can see that arrows turned into dotted lines. This architecture is completely asynchronous
    
* easy to implement: all integrations you need are native. You need just to configure serverless services and to implement a SQS consumer in your application.
    
* very scalable: you can add as many task as you want without affecting the database, your limit here is SNS but is very high. As stated in [official docs](https://docs.aws.amazon.com/general/latest/gr/sns.html) a single topic supports up to 12,500,000 subscriptions.
    
* no waste of resources: a.k.a really cost-effective. This solution leverages on pay-per-use services, and they would be used only when actual changes occurs on db.
    
* very easy to monitor: both SNS and SQS supports Dead Letter Topic / Queue: if a message isn't consumed within the timeout, it can be moved into a DLQ. You can set up an alarm if a DLQ is not empty, and kill the associated task.
    
* easy to recover: If a container cannot consume a message, it can try again. In other words, it does not have to be online and ready to receive the message at the moment it is delivered, as the queues are persistent.
    
* very fast: i did a benchmark on this solution, [here the github repo with the actual code](https://github.com/ncremaschini/fargate-notifications). Later on in this article we'll see results
    

### Cons

* more moving parts: even if the integration code is not required since it's provided by AWS, connecting things and tuning connections is not straightforward as performing a query.
    
* not so easy to troubleshoot. As every distributed system, i would say.
    
* it strongly depends on serverless services: if one link in the chain slows down or are not available, your containers can't be notified. We have to say that all involved services have a very good SLA: [3 nines for SQS and SNS](https://aws.amazon.com/it/messaging/sla/) and [4 nines for DynamoDB](https://aws.amazon.com/it/dynamodb/sla/). Not sure about Dynamo stream, since it appears to be not included in DynamoDB SLA. I suppose dynamo streams are backed by Kinesis Streams, [which also has 3 nines of availabilit](https://aws.amazon.com/it/kinesis/sla/)y.
    

### Open points:

The main open point here, to me, was: is this fast enough? Let's verify it.

## Trust, but verify

I couldn't find any official SLA about latency for involved services nor any AWS official benchmark.

So i decided to perform one myself, and i scripted a basic application using typescript and CDK / SDK.

[Here the github repo with the actual code](https://github.com/ncremaschini/fargate-notifications) and details on how the system is implemented.

Before going ahead, bare in mind that i performed this benchmark with the goal to understand if this combination of services / configuration could fit for my specific context / use case. Your context may be different, and this configuration may not fit with it.

### System design and data flow

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1709500848988/283edabf-0a85-43b3-bff1-22a8488a3aee.jpeg align="center")

1. The AppSync API receives mutations and stores derived data in the DynamoDB table
    
2. The DynamoDB stream the events
    
3. The Lambda function is triggered by the DynamoDB stream
    
4. The Lambda function sends the events to the SNS topic
    
5. The SNS topic sends the events to the SQS queues
    
6. The Fargate service reads the events from the SQS queues
    
7. If events are not processed within a timeout, they are moved to the DLQ
    
8. A Cloudwatch alarm is triggered if the DLQ is not empty
    

### Key system parameters:

* Region: eu-south-1
    
* Number of tasks: 20
    
* Event bus: 1 SQS per task, 1 DLQ per SQS, all SQS subscribed to one SNS
    
* SQS Consumer: provided by AWS SDK, configured for long polling (20s)
    
* Task configuration: 256 CPU, 512 Memory, Docker image based on [Official Node Image 20-slim](https://hub.docker.com/layers/library/node/20-slim/images/sha256-80c3e9753fed11eee3021b96497ba95fe15e5a1dfc16aaf5bc66025f369e00dd?context=explore)
    
* DynamoDB Configured in PayPerUseMode, stream enabled to trigger Lambda
    
* Lambda stream handler written in node20 bundled with [ESBuild](https://esbuild.github.io/), configured with 128MB
    

### Benchmark parameters

I used a basic postman collection runner to perform a mutation to Appsync every 5 seconds, for 720 iterations.

![postman runner execution recap](https://cdn.hashnode.com/res/hashnode/image/upload/v1708963551408/55810eb0-a306-4dec-9b96-2d2667fd8e19.png align="center")

### Goal

The goal was to verify if containers would be updated within 2 seconds.

### Measures

i used the following Cloudwatch provided metrics:

* Appsync latency
    
* Lambda latency
    
* Dynamo stream latency
    

and i created two custom metrics for measuring SQS and SNS time taken.

Time taken custom metrics are calculated from the SNS and SQS provided attributes:

* SNS Timestamp: [from AWS doc](https://docs.aws.amazon.com/sns/latest/dg/sns-message-and-json-formats.html)
    

> The time (GMT) when the notification was published.

* ApproximateFirstReceiveTimestamp: [from AWS doc](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_ReceiveMessage.html)
    

> returns the time the message was first received from the queue (epoch time in milliseconds).

* SentTimestamp: [from AWS doc](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_ReceiveMessage.html)
    

> Returns the time the message was sent to the queue (epoch time in milliseconds).

The following code snippet shows you how attributes are used to calculate *sns time taken in millis* and *sqs time taken in millis*

```typescript
      
//despite the name, this is the ISO Date the message was sent to the SNS topic
let snsReceivedISODate = messageBody.Timestamp;
if (snsReceivedISODate && message.Attributes) {   
   clientReceivedTimestamp = +message.Attributes.ApproximateFirstReceiveTimestamp!;
   sqsReceivedTimestamp = +message.Attributes.SentTimestamp!;
        
   let snsReceivedDate = new Date(snsReceivedISODate);
   snsReceivedTimestamp = snsReceivedDate.getTime();
   clientReceivedDate = new Date(clientReceivedTimestamp!);
   sqsReceivedDate = new Date(sqsReceivedTimestamp!);
        
   snsTimeTakenInMillis = sqsReceivedTimestamp - snsReceivedTimestamp;
   sqsTimeTakenInMillis = clientReceivedTimestamp - sqsReceivedTimestamp;
```

i didn't calculate the time taken by the client to parse the message because it really depends on the logic the client applies parsing the message.

### Results

Here screenshots from my Cloudwatch dashboard

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1709306099546/085f9eb4-f679-4165-9464-280cd4038f95.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708964741699/03618119-0de1-4d02-be44-fbebecb758f3.png align="center")

Few key data, from Average numbers:

* Most of time is taken by Appsync, i couldn't do anything to lower this latency since i used native Appsync native integration with DynamoDB.
    
* The only custom code is the Lambda stream processor code, and lamba duration is the second slowest component here. As you can see in the graph, the lambda cold start is the killer, but considering this we can observe a very good latency on avg (38 ms).
    
* The average total time taken is **108.39 ms**
    
* The average response time measured by my client, that cover my client network latency, is 92 ms. Given Appsync AVG Latency is 60.5 ms, my Avg network latency is 29.5 ms. This means that from my client sending the mutation to consumers receiving the message there are 108.39 + 29.5 = **137.89 ms**
    

### Conclusion

This solution has proven to be fast and reliable and requires little configuration to set up.

Since almost everything is managed, there is little space for tuning and improvements. In this particular configuration, I could simply give the Stream Processor Lambda more memory, but memory and latency do not scale (inversely) together.

I could remove Lambda and replace it with Event Bridge Pipe. I haven't tried it yet, but i'm going to use the exact same benchmark and compare the results.

Last but not least, keep in mind that AWS does not always include latency in the service SLA. I've run this benchmark a few times with comparable results, but I can't be sure that I would always get the same results over time. If your system requires stable and predictable performance over time, you can't go with services that don't include performance metrics in their SLA. You're better off taking control of the layers below, which means [you should consider going to a restaurant or even making your own pizza at home.](https://engineering.dunelm.com/pizza-as-a-service-2-0-5085cd4c365e)

## Wrap up

In this article, I have presented you with a solution that I had to design as part of my work and my approach to solution development: this includes clarifying the scope and context, evaluating different options and having a good knowledge of the parts involved and the performance and quality attributes of the overall system, writing code and benchmarking where necessary, but always with the clear awareness that there are no perfect solutions.

I hope it was helpful for you.

Bye 👋!