---
title: Parallelize independent I/O without parallelizing Spring JDBC DB access
tags: [springboot-kotlin, coroutines, transactions, jdbc]
scope: cross-project
status: draft
---

# Parallelize Independent I/O Without Parallelizing Spring JDBC DB Access

## Context
Use this when a Spring Boot Kotlin service has both blocking JDBC/jOOQ reads and an independent non-DB I/O call, such as a Feign or WebClient request. Spring JDBC transaction state is thread-bound, so DB work that belongs to one logical read flow should stay inside one transaction callback.

## Wrong Direction
Do not put separate DB selects into separate `async` blocks or call coroutine fan-out utilities inside a `TransactionTemplate` callback. That can move DB access across threads and transactions, and it makes read consistency and transaction routing harder to reason about.

## Correct Pattern
Group DB reads that can share one transaction into a single transaction-port call, then run that whole DB-read unit in parallel only with independent non-DB I/O.

```kotlin
val (externalResult, databaseReads) = asyncAndAwait(
    { externalClient.findAll() },
    {
        transactionalPort.executeReadOnlyAndReturn(
            { firstDbSelect() },
            { secondDbSelect() },
        )
    },
)
```

## Reusable Insight
The parallelism boundary should be around the complete DB transaction, not inside it. Non-DB I/O and DB I/O can overlap when they are independent, but DB access itself should remain sequential within the transaction.

## Detection
Look for services that call `transactionalPort.executeReadOnly` multiple times in one flow or call DB ports from multiple `async` blocks. If the DB reads are part of one logical snapshot, merge them into one transaction call before adding coroutine parallelism.

## Verification
Add a test double for the transaction port that counts `executeReadOnly` invocations. Add virtual-time coroutine tests with equal delays on the non-DB call and the DB transaction unit: parallel execution should finish in one delay interval, while the transaction count should remain one.
