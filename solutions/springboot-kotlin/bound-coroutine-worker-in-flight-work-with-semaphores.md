---
title: Bound coroutine worker in-flight work with semaphores
tags: [springboot-kotlin, coroutines, worker, concurrency]
scope: cross-project
status: draft
---

# Bound coroutine worker in-flight work with semaphores

## Context
Use this when a Spring Boot Kotlin worker processes a claimed batch with
coroutines and each item performs suspending I/O such as WebClient calls,
message sends, or delayed polling.

## Wrong Direction
Do not rely on `Dispatchers.Default.limitedParallelism(n)` to cap the number of
in-flight business jobs when the job body suspends. `limitedParallelism` limits
concurrently executing coroutine work on that dispatcher. Once a coroutine
suspends on nonblocking I/O or `delay`, its dispatcher slot can be reused and
more jobs can enter the business section.

## Correct Pattern
Use a coroutine `Semaphore` and hold the permit across the whole item lifecycle:

```kotlin
class CoroutineConcurrencyLimiter(
    permits: Int,
    private val dispatcher: CoroutineDispatcher = Dispatchers.Default,
) {
    private val semaphore = Semaphore(permits)

    suspend fun <T> process(
        items: Collection<T>,
        block: suspend (T) -> Unit,
    ) {
        coroutineScope {
            items.map { item ->
                async(dispatcher) {
                    semaphore.withPermit {
                        block(item)
                    }
                }
            }.awaitAll()
        }
    }
}
```

Create the limiter once for the worker/service lifecycle and reuse it across
batch calls:

```kotlin
private val workLimiter = CoroutineConcurrencyLimiter(concurrency)

suspend fun processDueJobs(batchSize: Int) {
    val claimed = claim(batchSize)
    workLimiter.process(claimed) { job ->
        process(job)
    }
}
```

Avoid constructing the semaphore inside the batch helper:

```kotlin
private suspend fun <T> Collection<T>.processBounded(
    block: suspend (T) -> Unit,
) {
    coroutineScope {
        val semaphore = Semaphore(concurrency)
        map { item ->
            async(Dispatchers.Default) {
                semaphore.withPermit {
                    block(item)
                }
            }
        }.awaitAll()
    }
}
```

That pattern only bounds a single method invocation. If multiple worker loops
or overlapping calls run on the same service instance, each call gets a separate
permit pool and the real in-flight count can exceed the configured worker
concurrency.

Keep blocking JDBC or jOOQ work inside the project's transaction executor or
virtual-thread-backed dispatcher. The semaphore should bound the logical job
flow, not replace the DB transaction execution policy.

## Reusable Insight
Dispatcher parallelism is a thread execution limit; a semaphore is an in-flight
workflow limit. Suspended coroutines free dispatcher capacity but should often
still count against worker concurrency. The semaphore itself is also the
limiter state, so keep it at the same lifecycle as the concurrency budget rather
than allocating it per batch.

## Detection
Look for tests where `maxInFlight` exceeds the configured concurrency even
though the implementation uses `limitedParallelism`. Also look for worker code
that launches one coroutine per claimed row and relies only on dispatcher choice
to enforce downstream API pressure. Review helpers that call `Semaphore(n)`
inside every batch invocation; prefer a reusable limiter field on the worker or
service.

## Verification
Write a test that increments an `inFlight` counter before the suspending call,
delays or waits, decrements after completion, and asserts the maximum observed
in-flight count equals the configured concurrency. Include more jobs than the
limit so the test fails if every coroutine can enter the job body at once.
