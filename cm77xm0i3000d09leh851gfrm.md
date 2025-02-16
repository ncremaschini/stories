---
title: "Building Atomic Counters with TiDB"
seoTitle: "Create Atomic Counters Using TiDB"
seoDescription: "Learn about atomic counters in TiDB for consistent, highly available distributed systems with examples and implementation tips"
datePublished: Sun Feb 16 2025 18:00:04 GMT+0000 (Coordinated Universal Time)
cuid: cm77xm0i3000d09leh851gfrm
slug: building-atomic-counters-with-tidb
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/f7YQo-eYHdM/upload/4652898d884ef17c74e41f6fc4cfbdb7.jpeg
tags: aws, sql, distributed-system, serverless, newsql, tidb

---

Distributed SQL databases have become a cornerstone for applications that require **global scalability and strong consistency**, and this problem has existed since the very first deployment of a database on two distinct servers: how to achieve strong consistency and scalability without compromising availability?

Is it possible?

The [CAP theorem](https://en.wikipedia.org/wiki/CAP_theorem) states that this isn't possible: if you consider consistency, availability and partitioning as fundamental properties of data storage, you can only choose two out of three properties.

In this fourth part of the series on atomic counters, we'll explore how the pattern can be implemented using [TiDB](https://www.pingcap.com/tidb-cloud-serverless/), [NewSql's](https://en.wikipedia.org/wiki/NewSQL) database class, as an example, focusing on global partitioning, strong consistency and high availability.

This article will provide a closer look at TiDB’s unique architecture, discuss trade-offs, and refer to a practical implementation found in [this GitHub repository](https://github.com/ncremaschini/atomic-counter).

# **Serializability, Linearizability, and TiDB**

TiDB guarantees **strong consistency** across its distributed nodes by adopting a **two-phase commit (2PC)** protocol. This ensures that all transactions, including atomic increments, are serialized and linearizable.

To provide high availability, TiDB replicates data using [**Raft**](https://en.wikipedia.org/wiki/Raft_\(algorithm\)), a consensus algorithm that ensures data consistency across regions. This makes TiDB well-suited for use cases requiring globally consistent counters.

# **Replication and Leader Election in TiDB**

TiDB’s replication model is built on **Raft**, where each region has a leader and multiple followers.

• The **Raft leader** handles writes and ensures consistency through consensus.

• Followers replicate data for high availability and enable failover in case of leader failure.

This replication mechanism ensures that even in multi-region deployments, TiDB maintains consistency and availability.

# **The Atomic Counter Pattern**

The **atomic counter pattern** ensures precise, consistent counter increments even in distributed environments.

With TiDB, you can achieve this using **SQL transactions** and **atomic operations** like UPDATE ... SET.

# **Hands-On! Walkthrough of the Deployable Example**

Let’s examine how the deployable example in [this GitHub repository](https://github.com/ncremaschini/atomic-counter) implements an atomic counter using **AWS Lambda**, **API Gateway**, and **TiDB**.

* **Api Gateway:** Provides HTTP endpoints for interacting with the counter.
    
* **Lamba function:** Implements the business logic for incrementing the counter.
    
* **TiDB:** Stores the counters.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1739721641453/db024e84-f5c0-4d2e-8d1b-d09cb8ae9474.png align="center")

In my example project you can decide wheter to use a maximum value for the counter or not: this determine if use or not conditional writes.

Let’s focus on lambda business logic, **from the** [tiDBAtomicCounter Lambda code:](https://github.com/ncremaschini/atomic-counter/blob/main/lib/lambda/tiDB/counterLambda/index.ts)

```typescript
    connection = await createDbConnection(DB);

    const updateFilter = getUpdateFilter(useConditionalWrites);

    const params = {
      id: id,
      max_value: maxCounterValue
    }

    const [rows] = await connection.query<RowDataPacket[]>(updateFilter, params);
```

The *getUpdateFilter(useConditionalWrites)* method provides a specific filter based on *useConditionalWrites* boolean variable.

The method is very simple, and i kept it a little verbose than required just for better comprehension.

It basically returns one of two static SQL statements, very similar to each other, with a little but really important difference:

```typescript
const getUpdateFilter = (useConditionalWrites: boolean): string => {

  const unconditionalWriteParams = 'SELECT counter_value FROM counters WHERE counter_id = :id FOR UPDATE;  \
                                    INSERT INTO counters (counter_id, counter_value) VALUES (:id, 1) \
                                    ON DUPLICATE KEY UPDATE counter_value = counter_value + 1; \
                                    SELECT counter_value FROM counters WHERE counter_id = :id; \
                                    COMMIT;';

  const conditionalWriteParams = 'SELECT counter_value FROM counters WHERE counter_id = :id FOR UPDATE;  \
                                  INSERT INTO counters (counter_id, counter_value) VALUES (:id, 1) \
                                  ON DUPLICATE KEY UPDATE counter_value = IF(counter_value < :max_value, counter_value + 1, counter_value);\
                                  SELECT counter_value FROM counters WHERE counter_id = :id; \
                                  COMMIT;';

  return useConditionalWrites ? conditionalWriteParams : unconditionalWriteParams;
}
```

Let’s break down each SQL statements:

## First statement

```sql
SELECT counter_value FROM counters WHERE counter_id = :id FOR UPDATE; 
```

This statement tells to the db engine that you are selecting the specific table row for update.

Based on the db engine lock mechanism, it locks the row for the duration of the transascion:

* Pessimistic locking: immediately when executing the statement, preventing other concurrent transactions from modifying it.
    
* Optmistic locking: lock is not acquired immediately, but the engine would check for conflicts at commit time and retries if conflicts occur.
    

TiDB default has changed over time, and it is [configurable](https://docs.pingcap.com/tidb/stable/pessimistic-transaction).

I suggest to consider carefully tradeoffs between the two modes: the right one really depends on your specific use case.

[Knowledge is free at the library. Just bring your own container.](https://docs.pingcap.com/tidb/stable/transaction-overview)

## Second statement

```sql
INSERT INTO counters (counter_id, counter_value) VALUES (:id, 1)
```

Nothing special, just tells the db to insert the new row.

But, wait: we were supposed to talk about incrementing counters, not to inserting new rows!

The third statement is where the magic happens.

## Third statement (unconditional writes):

```sql
ON DUPLICATE KEY UPDATE counter_value = counter_value + 1;
```

This statement tells to db engine what to do if the previous statement fails for duplicate key, because we are trying to insert two rows with the same *counter\_id* wich is the table’s primary key.

We are basically asking:

> please increment by one the counter\_value field of the row

With the second and the third statement togheter, we are telling to the db engine:

> please insert this new counter, but if the counter is already present, don’t panic and just increment it

## Third statement, with conditional writes:

```sql
ON DUPLICATE KEY UPDATE counter_value = IF(counter_value < :max_value, counter_value + 1, counter_value);
```

just as the unconditional write version, we are telling the db engine to increment the existing row but only if the current counter value is below the *max\_value* parameter.

> please insert this new counter, but if the counter is already present, don’t panic and just increment it if it is below the max value.

## Fourth statement

```sql
SELECT counter_value FROM counters WHERE counter_id = :id
```

This is just for retrevieng of the value after the insert / update.

## Final statement

```sql
COMMIT;
```

This seems like the simplest statement, but this is where all the magic happens: based on your DB Engine configuration, this is where our five-statement transaction is executed atomically on the server, conflicts are solved and then data replicated if it was successful.

Since transactions are executed on the server and are all-or-nothing statements, they fit the atomic counter pattern perfectly.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1739727388573/be4abe2a-3608-4366-bcf5-77015e695556.png align="center")

# **Trade-Offs and Conclusion**

## **Strengths**:

• **Strong Consistency**: TiDB’s Raft-based replication and 2PC protocol ensure consistent increments even in globally distributed environments.

• **SQL Familiarity**: Developers can use familiar SQL syntax, reducing the learning curve.

## **Limitations**:

• **Latency**: Cross-region communication for strong consistency may increase latency, especially with pessimistic locking configuration.

• **Operational Complexity**: While TiDB Cloud simplifies management, understanding distributed SQL concepts is necessary for effective use.

## **Key Takeaway**:

TiDB is an excellent choice for globally distributed applications requiring strong consistency. Its support for SQL transactions and automatic scaling makes it a powerful tool for implementing atomic counters in multi-region setups.