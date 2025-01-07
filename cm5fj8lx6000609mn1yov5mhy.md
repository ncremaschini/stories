---
title: "Building Atomic Counters with Momento"
seoTitle: "Atomic Counters with Momento Guide"
seoDescription: "Learn to implement atomic counters using Momento, a serverless caching solution, and compare it with Redis for simplicity and scalability"
datePublished: Thu Jan 02 2025 16:20:28 GMT+0000 (Coordinated Universal Time)
cuid: cm5fj8lx6000609mn1yov5mhy
slug: building-atomic-counters-with-momento
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/9Njoam3Vesc/upload/3f2ce5b080d59a789893686f19cd547f.jpeg
tags: aws, databases, caching, distributed-system, consistency

---

In the world of distributed systems, **serverless caching** is gaining traction for its simplicity and scalability. [**Momento**](https://www.gomomento.com/), a fully managed serverless cache, builds on the core concepts of caching while eliminating infrastructure management.

In this third installment of the [atomic counter series](https://haveyoutriedrestarting.com/series/atomic-counter), we’ll explore how to implement the pattern using [**Momento**](https://www.gomomento.com/). By comparing it to [**Redis**](https://redis.io/), we’ll highlight how Momento simplifies caching for developers, discuss its trade-offs, and guide you through a practical implementation using the code in [this GitHub repository](https://github.com/ncremaschini/atomic-counter).

# **Serializability, Linearizability, and Momento**

Unlike traditional caching systems, Momento operates as a **serverless service**, meaning you don’t manage nodes, replicas, or clusters.

However, like Redis, it provides atomic operations such as increment.

Momento’s atomicity ensures that counter updates are serialized within its storage layer. However, consistency across distributed systems can vary based on use cases, which aligns with the **eventual consistency** model in serverless architectures.

# **Replication and Leader Election in Momento**

As a managed service, **Momento abstracts replication and failover**.

You don’t have visibility into specific replicas or leaders, but the platform ensures high availability by handling replication and redundancy under the hood.

This is a notable difference from Redis, where you control and configure replication explicitly.

Momento offers simplicity at the cost of operational transparency and fine-grained control.

# The Atomic Counter pattern

The **atomic counter pattern** enables precise increment operations, even in distributed environments.

With Momento, you use its increment operation, which automatically initializes the counter if it doesn’t exist, similar to Redis.

This approach works very well if you need to increment your counter unconditionally, regardless its current value.

If you need to implement conditional increment, Momento doesn’t provide methods to do so on server side: you have to handle it on client side, and this could lead to race conditions.

# **Hands-on! Walkthrough of the Deployable Example**

Let’s examine how the deployable example in [this GitHub repository](https://github.com/ncremaschini/atomic-counter)

This example demonstrates how to implement an atomic counter using **AWS Lambda**, **API Gateway**, and **Momento**.

1\. **API Gateway**: Provides HTTP endpoints for interacting with the counter.

2\. **Lambda Functions**: Implements the business logic for incrementing the counter.

3\. **Momento**: Stores the counters.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1736244695494/c95ab48e-12fd-4bf0-84e1-87b2e284fbae.png align="center")

In my example project you can decide wheter to use a maximum value for the counter or not: this determine if use or not conditional writes.

Let’s focus on lambda business logic, **from the** [**momentoAtomicCounter**](https://github.com/ncremaschini/atomic-counter/blob/main/lib/lambda/momento/index.ts) **Lambda** code:

```typescript
const momentoCacheClient = await buildMomentoClient();

let counter = 0;

if (useConditionalWrites) {
    counter = await handleConditionalWrites(momentoCacheClient,cacheName, id, maxCounterValue);
 } else {
    counter = await handleUnconditionalWrites(momentoCacheClient,cacheName, id);
 }
```

As you can see, i wrote two different methods to handle conditional and unconditional writes.

Let’s dive into the simpler one, *handleUnconditionalWrites*:

```typescript
async function handleUnconditionalWrites(momentoClient: CacheClient,cacheName: string, id: string) {
  let counter = 0;

  const cacheIncrementResponse = await momentoClient.increment(cacheName, id, 1);
  switch (cacheIncrementResponse.type) {
    case CacheIncrementResponse.Success:
      counter = cacheIncrementResponse.value();
      break;
    case CacheIncrementResponse.Error:
      throw new Error(cacheIncrementResponse.message());
  }

  return counter
}
```

The method simply levearage on the *increment* method from momento sdk.

It increments a key value by an integer (one, in this example) regardless key existence or key current value.

Things get more interesting when it comes to handle conditional writes to increment the counter only if it is below a specified threshold.

Momento does not provide any conditional increment method, but provides few useful conditional writes methods such *setIfPresentAndNotEqual* and *setIfAbsent.*

Let’s dive into *handleContionalWrites* implementation (response handling logic is removed better readability):

```typescript
async function handleConditionalWrites(momentoClient: CacheClient, cacheName: string, id: string, maxCounterValue: string){

  let counter = 0;

  const cacheGetResponse = await momentoClient.get(cacheName, id);

  switch (cacheGetResponse.type) {
    case CacheGetResponse.Hit:
      const currentCounter = Number(cacheGetResponse.value());
      const nextCounter =  currentCounter + 1;
      const strNextCounter = nextCounter.toString();

      counter = await handleSetIfPresentAndNotEqual(momentoClient,cacheName, id, strNextCounter, maxCounterValue);
      break;  
    case CacheGetResponse.Miss:
      counter = await hanldeSetIfAbsent(momentoClient, cacheName, id, '1');
      break;
    case CacheGetResponse.Error:
      throw new Error(cacheGetResponse.toString()); 
  }

  return counter
}

async function handleSetIfPresentAndNotEqual(momentoClient: CacheClient,cacheName: string, id: string, nextCounter: string, maxCounterValue: string) {
  
  const cacheSetIfPresentAndNotEqualResponse = await momentoClient.setIfPresentAndNotEqual(cacheName, id, nextCounter, maxCounterValue);
  ...
}

async function hanldeSetIfAbsent(momentoClient: CacheClient,cacheName: string, id: string, value: string) {
  let counter = 0;
  const setIfAbsentResponse = await momentoClient.setIfAbsent(cacheName, id, value);
  ...
}
```

These methods perform the following logic:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1736244822141/c7e38cd1-24f8-4ecb-9da4-20880b7c068a.png align="center")

and this ensures consistency, since race conditions are checked by the two check-and-set methods on server-side.

This is how key initialization works (*set if not present* branch):

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1736245870347/f55ef434-e011-4de1-85fd-e9022fa5bb3e.png align="center")

and this is how an existing key increment works (*set if present and not equals* branch):

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1736245944886/ddf6c0c1-fd30-4134-aa68-36600c849d0a.png align="center")

# Trade-Offs and conclusion

## Strenghts:

* **Serverless Simplicity**: No infrastructure to manage, reducing operational overhead.
    
* **Built-In Scalability**: Automatically scales to meet demand without manual intervention.
    

## Limitations:

* **Reduced Control**: Lack of visibility into replication and cluster configurations.
    
* **Eventual Consistency**: While atomic operations are supported, consistency guarantees may differ in highly distributed setups.
    
* Performance: since an initial GET is required, more network trips are required compared to other solution that implements conditional increments.
    

# Conclusion

Momento showcases how a **serverless-first approach** simplifies distributed caching.

By eliminating the need to manage infrastructure, it allows developers to focus on building applications rather than worrying about operational overhead.

For atomic counters, Momento’s increment operation makes implementation straightforward and reliable. However, this convenience comes with trade-offs: you lose the granular control over replication and failover configurations that traditional systems like Redis offer.

If you’re exploring distributed counters for your application, I highly recommend trying out the example provided in the [GitHub repository](https://github.com/ncremaschini/atomic-counter).

Stay tuned for the next installment in the series, where we’ll delve into **DocumentDB**,