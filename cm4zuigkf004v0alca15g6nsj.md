---
title: "Building Atomic Counters with Elasticache Redis"
seoTitle: "Atomic Counters with Elasticache Redis"
seoDescription: "Learn to build atomic counters with Redis on AWS ElastiCache. Explore replication, atomic operations, and conditional writes in this guide"
datePublished: Sun Dec 22 2024 16:51:45 GMT+0000 (Coordinated Universal Time)
cuid: cm4zuigkf004v0alca15g6nsj
slug: building-atomic-counters-with-elasticache-redis
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/gdL-UZfnD3I/upload/1ab757f37fb175ede5dfb1e78f04fb41.jpeg
tags: cloud, aws, redis, databases, distributed-system, serverless

---

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">Please read this article in DARK mode or you won’t see diagrams lines</div>
</div>

When working with high-throughput, low-latency applications, **Redis**—an in-memory data store—stands out as an excellent choice for implementing the **atomic counter pattern**.

With its atomic operations and simple APIs, Redis offers a straightforward approach to incrementing counters while ensuring high performance.

In this article, we’ll explore how to build an atomic counter using **AWS ElastiCache Redis**.

You’ll gain a practical understanding of Redis concepts like **atomic operations**, its **replication model**, and how to implement counters with the code from [this GitHub repository](https://github.com/ncremaschini/atomic-counter).

## **Serializability, Linearizability, and Redis**

Before we can dive deep into code, we need to recall few concepts (please refer to [**the first article of this series for a detailed explanation**)](https://hashnode.com/post/cm3syajxr000009mk7pwz56if):

* **Serializability**: Operations appear in a consistent sequential order, ensuring correctness.
    
* **Linearizability**: Writes are immediately visible for subsequent reads, ensuring real-time consistency.
    

Redis processes commands in a **single-threaded event loop**, ensuring that each command is executed in the order it’s received. This guarantees atomicity at the command level for operations like *INCR*.

While Redis operations on a single node can be considered **linearizable**, in a distributed Redis setup (e.g., with clustering or replicas), this strict ordering can break. Writes to replicas are propagated asynchronously, so they may lag behind the primary node.

## **Replication and Leader election in Redis**

Redis employs a **primary-replica architecture**, where:

* The **primary node** handles all writes and propagates updates to replicas asynchronously.
    
* **Redis Sentinel** handles failover, promoting a replica to primary in case of failure.
    

For atomic counters, a single primary node is typically sufficient. If clustering is used, counter keys should be kept on a single shard to maintain atomicity.

## The Atomic Counter Pattern

The **atomic counter pattern** allows you to increment a value reliably, even in distributed systems, by ensuring operations are conflict-free and consistent.

Redis supports this pattern natively through the *INCR* command, which atomically increments a key’s value by 1.

However, if it is necessary to carry out the increment depending on the current status of the counter, race conditions may be possible.

## **Hands-on! Walkthrough of the Deployable Example**

Let’s dive into the example provided in the [GitHub repository](https://github.com/ncremaschini/atomic-counter).

This example demonstrates how to implement an atomic counter using **AWS Lambda**, **API Gateway**, and **ElastiCache Redis**.

1\. **API Gateway**: Provides HTTP endpoints for interacting with the counter.

2\. **Lambda Functions**: Implements the business logic for incrementing the counter.

3\. **ElastiCache Redis** Cluster: Stores the counters with atomicity guarantees.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1734792672394/4124f58b-1234-47df-a950-af2e642adfcc.png align="center")

In my example project you can decide wheter to use a maximum value for the counter or not: this determine if use or not conditional writes.

Let’s focus on lambda business logic, [from the redisAtomicCounter Lambda](https://github.com/ncremaschini/atomic-counter/blob/main/lib/lambda/redis/index.ts) code:

```typescript
const redisClient = await buildRedisClient();

const result = await redisClient.eval(getLuaScript(useConditionalWrites), 1, id, maxCounterValue);
```

This code snippet simply sends an *eval* command and gets the new updated counter value, using the [ioredis client](https://github.com/redis/ioredis).

Le’t see how the *getLuaScript()* works:

```typescript
const getLuaScript = (useConditionalWrites: boolean) => {

  const unconditionalIncrementScript = `
    redis.call('INCR', KEYS[1])
    local counter = redis.call('GET', KEYS[1])
    return counter
   `;

  const conditionalIncrementScript = `
 
    local counter = redis.call('GET', KEYS[1])
    local maxValue = tonumber(ARGV[1])
    
    if not counter then
      counter = 0
    end
    
    counter = tonumber(counter)

    if counter < maxValue then
      redis.call('INCR', KEYS[1])
      counter = redis.call('GET', KEYS[1])
      return counter
    else
      return 'Counter has reached its maximum value of: ' .. maxValue
    end
  `;

  return useConditionalWrites ? conditionalIncrementScript : unconditionalIncrementScript;
}
```

Unconditional writes don't require actually to be executed inside a LUA Script: we could use the *incr()* method provided by *ioredis* client.

But for conditional writes, it is required to check the counter value before incrementing it to avoid race conditions: if we perform the check on client side, another client might increment the counter between the *get()* and the *incr()* instructions execution on the first client.

Let's see an example: assuming the maximum value for the counter is 10, Alice and Bob perform a *GET* for the same key 1, when the counter value is 9.

They check if the current value is below the maximum value, and then they both send an *INC* command to redis.

Since *INC* command is executed unconditionally on server side, the counter is incremented by two and exceeds the maximum value.

![Performing checks on client side can lead to race contisions](https://cdn.hashnode.com/res/hashnode/image/upload/v1734871735176/081f2c3a-d489-4fa4-8755-4432389b5e54.png align="center")

## Redis secret sauce: LUA Script

The solution is to check the counter value on server side, executing a LUA Script.

It gets the counter value, check if it is below the maximum value and eventually increment it:

```julia

local counter = redis.call('GET', KEYS[1])
local maxValue = tonumber(ARGV[1])
    
if not counter then
    counter = 0
end
    
counter = tonumber(counter)

if counter < maxValue then
    redis.call('INCR', KEYS[1])
    counter = redis.call('GET', KEYS[1])
    return counter
else
    return 'Counter has reached its maximum value of: ' .. maxValue
end
```

If you look the code, you might think this code is not safe too: another script could increment the counter between the *GET* and the *INCR* command execution.

The magic of LUA Script is that just one script can be executed at the same time, [as reported in the documentation](https://redis.io/docs/latest/develop/interact/programmability/eval-intro/):

> Redis guarantees the script's atomic execution. While executing the script, all server activities are blocked during its entire runtime. These semantics mean that all of the script's effects either have yet to happen or had already happened.

and that is exactly what we need when dealing with atomic counters: Since only one script is executed, there is no concurrency and no race conditions, as **serializability is** guaranteed.

![LUA Script avoids race conditions](https://cdn.hashnode.com/res/hashnode/image/upload/v1734871750087/d2ffa87f-767f-4020-802c-a121d71b7fc0.png align="center")

Moreover, we have better performance, which is good:

> Because scripts execute in the server, reading and writing data from scripts is very efficient.

## **Trade-Offs and Conclusion**

### **Strengths**:

* Redis provides **low-latency** atomic operations.
    
* The *INCR* command is inherently atomic, simplifying counter implementation.
    
* LUA Script execution prevent race conditions on conditional writes, achieving **serializability.**
    

### **Limitations**:

* **Asynchronous Replication**: Updates may not immediately reflect on replicas.
    
* **Durability Risks**: Without persistence, counters may reset after a failure or restart.
    

### **Conclusion:**

Redis is ideal for high-performance, in-memory atomic counters where latency is a top priority, and simplifies building atomic counters with its native support for atomic operations and low-latency access.

However, consider its replication and durability trade-offs for production use.

Check out the [deployable example here](https://github.com/ncremaschini/atomic-counter), and stay tuned for the next article in the series, where we’ll explore **Momento** as an alternative for serverless caching.