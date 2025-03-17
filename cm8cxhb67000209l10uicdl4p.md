---
title: "Building Atomic Counters with Amazon DocumentDB"
seoTitle: "Atomic Counters with Amazon DocumentDB"
seoDescription: "Learn how to implement atomic counters using Amazon DocumentDB for consistent, scalable operations in distributed environments"
datePublished: Mon Mar 17 2025 10:34:57 GMT+0000 (Coordinated Universal Time)
cuid: cm8cxhb67000209l10uicdl4p
slug: building-atomic-counters-with-amazon-documentdb
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/nsQeVhtnyFc/upload/1139784b30fc09822eddc2a56176210d.jpeg
tags: cloud, aws, mongodb, databases, distributed-system, serverless, aws-documentdb

---

# Introduction

This is the final installment in my [atomic counters series](https://haveyoutriedrestarting.com/series/atomic-counter) where I explore different distributed databases and how they implement atomic counters.

This time, we’re looking at [Amazon DocumentDB](https://aws.amazon.com/documentdb/) a managed NoSQL document database, [**MongoDB-compatible**](https://www.mongodb.com/) and optimized for AWS.

Atomic counters are a common requirement in distributed applications, whether for tracking views, managing inventory, or implementing rate limiting.

In this article, we’ll discuss how **DocumentDB handles atomic updates** and explore a **working example** from my [GitHub repository.](https://github.com/ncremaschini/atomic-counter)

## **Serializability and Linearizability in DocumentDB**

Before we can dive deep into code, we need to recall few concepts (please refer to [**the first article of this series for a detailed explanation**)](https://haveyoutriedrestarting.com/atomic-counter-framing-the-problem-space)

* **Serializability**: Operations appear in a consistent sequential order, ensuring correctness.
    
* **Linearizability**: Writes are immediately visible for subsequent reads, ensuring real-time consistency.
    

DocumentDB achieves linearizable writes through its **single-primary, multi-replica architecture**:

• **Write operations are directed to the primary instance**, and changes are asynchronously replicated to secondaries.

• **Read operations from the primary always return the latest committed value**, ensuring linearizability.

• **Replica reads may return stale data** due to replication lag, meaning they are eventually consistent.

This guarantees that **atomic updates within a single document, like counters using the $inc operator, remain correct and isolated**.

While not required in this specific scenario, it is worth to mention DocumentDB supports:

* [read isolation level configuration](https://docs.aws.amazon.com/documentdb/latest/developerguide/how-it-works.html#durability-consistency-isolation)
    
* [transactions and their isolation level, read and write concerns configuration.](https://docs.aws.amazon.com/documentdb/latest/developerguide/transactions.html)
    

## Replication and Leader Election in DocumentDB

DocumentDB automatically replicates data across multiple availability zones to ensure durability and availability.

Key mechanisms include:

• **Single-primary replication**: A single primary instance handles writes, while replicas asynchronously replicate data and serve read requests.

• **Leader election**: If the primary instance fails, DocumentDB automatically promotes a replica to primary, minimizing downtime and maintaining availability.

This replication strategy allows DocumentDB to scale reads across replicas while ensuring that writes remain **strongly consistent** on the primary.

However, applications must account for **eventual consistency** when reading from replicas due to asynchronous replication.

# The Atomic Counter Pattern

The atomic counter pattern enables precise increment operations, even in distributed environments.

With DocumentDB, you use the **$inc operator**, which atomically increments a numeric field within a document.

This ensures that **concurrent increments are safely serialized** without race conditions.

**DocumentDB supports conditional increments natively**: you can use **$inc with $cond in an update operation** to increment the counter only when certain conditions are met—**all in a single atomic operation**.

This makes DocumentDB a good choice when you need **both unconditional and conditional increments**, ensuring correctness without requiring complex client-side logic.

# **Hands-on! Walkthrough of the Deployable Example**

Let’s examine how the deployable example in [**this GitHub repository**](https://github.com/ncremaschini/atomic-counter)

This example demonstrates how to implement an atomic counter using **AWS Lambda**, **API Gateway**, and **DocumentDB**.

1\. **API Gateway**: Provides HTTP endpoints for interacting with the counter.

2\. **Lambda Functions**: Implements the business logic for incrementing the counter.

3\. **DocumentDB**: Stores the counters.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1742054210547/4ff5e661-e4ff-4b02-8b75-c85ebe5df724.png align="center")

In my example project you can decide wheter to use a maximum value for the counter or not: this determine if use or not conditional writes.

Let’s focus on lambda business logic, **from the** [docDbCounterLambda](https://github.com/ncremaschini/atomic-counter/blob/main/lib/lambda/documentDB/counterLambda/index.ts) code:

```typescript
const documentDBClient = await buildDocumentDbClient();

await documentDBClient.connect();

const countersCollection = documentDBClient.db("atomic_counter").collection('counters');

const updateFilter = getUpdateFilter(useConditionalWrites, id, maxCounterValue);

const updateResult = await countersCollection.updateOne(
    updateFilter,
    {
        $inc: { atomic_counter: 1 }
    },
    {
        upsert: true, 
    }
);
```

Here I use the **$inc** operator with the *upsert* flag set to true: this makes the method work both the first time, when the counter does not exist and is therefore initialized to zero, and for further increment operations.

What changes between conditional and unconditional write operations is the *updateFilter* returned by the *getUpdateFilter* method.

Let’s have a look at it:

```typescript
const getUpdateFilter = (useConditionalWrites: boolean, id: number, maxCounterValue: string) => {
  const unconditionalWriteParams = {
    counter_id: id
  }

  const conditionalWriteParams = {
    counter_id: id,
    $and: [
      { atomic_counter: { $lt: Number(maxCounterValue) } }
    ],
  }

  return useConditionalWrites ? conditionalWriteParams : unconditionalWriteParams;
}
```

For unconditional writes, the only filter is the *counter\_id* attribute.

For conditional writes, the construct **$lt** (lower than) is added as an additional condition to check whether the value is below the maximum value.

Since the update is performed for a single document and the increment operation is performed on the server side, atomicity is guaranteed and the counter value cannot exceed the maximum value

![if two concurrrent increments are requested and the second one would exceed the maximum value, the first is accepted while the second is rejected](https://cdn.hashnode.com/res/hashnode/image/upload/v1742121822865/5599647f-2f8e-4ef2-a920-d55fa2de15aa.png align="center")

# **Trade-Offs and Conclusion**

Like other databases in this series, DocumentDB comes with **trade-offs** when used for atomic counters:

## Strenghts:

* **MongoDB Compatibility**: Developers familiar with MongoDB can reuse existing knowledge.
    
* **Managed Scaling**: AWS handles **replication, backups, and failover**.
    
* **Atomic Updates on a Single Document**: **$inc** ensures updates are atomic.
    

## Limitations:

* **Eventual Consistency for Replicas**: Secondary reads may return stale data.
    
* **Higher Latency for Stronger Consistency**: To ensure **fresh data**, queries must be sent to the primary instance.
    

## **Key Takeaways:**

* **Atomic counters in DocumentDB** can be implemented using the **$inc** operator, ensuring **atomic updates at the document level**.
    
* **Conditional increments are fully supported** using **$inc** combined with **$cond**, allowing for server-side enforcement of constraints.
    
* **DocumentDB follows a single-primary, multi-replica model**, meaning writes are **strongly consistent**, but replica reads may be **eventually consistent**.
    
* **Automatic leader election** ensures high availability by promoting a replica to primary in case of failure.
    

You can find the **full runnable example** in my GitHub repository: [atomic-counter](https://github.com/ncremaschini/atomic-counter).

This marks the end of the **atomic counter series**! 🚀