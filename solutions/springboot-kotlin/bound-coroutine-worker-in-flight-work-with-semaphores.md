---
title: Coroutine worker의 in-flight 작업은 semaphore로 제한한다
tags: [springboot-kotlin, coroutines, worker, concurrency]
scope: cross-project
status: active
principles: [persistence-transactions]
---

# Coroutine worker의 in-flight 작업은 semaphore로 제한한다

> 관련 공통 원칙: [영속성, 트랜잭션, 동시성 경계](../../principles/ko/persistence-transactions.md)

## 적용 시점

Spring Boot Kotlin worker가 claimed batch를 coroutine으로 처리하고, 각 item이 WebClient call, message send, delayed polling 같은 suspending I/O를 수행할 때 적용한다.

## 피해야 할 방향

`Dispatchers.Default.limitedParallelism(n)`만으로 in-flight business job 수가 제한된다고 믿으면 안 된다. `limitedParallelism`은 해당 dispatcher에서 동시에 실행 중인 coroutine work를 제한한다. coroutine이 nonblocking I/O나 `delay`에서 suspend되면 dispatcher slot은 다시 사용될 수 있고, 더 많은 job이 business section에 들어올 수 있다.

batch helper 안에서 매번 semaphore를 새로 만드는 것도 피한다. 그 방식은 한 method invocation만 제한한다. 여러 worker loop나 overlapping call이 같은 service instance에서 실행되면 호출마다 별도 permit pool이 생겨 실제 in-flight count가 configured worker concurrency를 넘을 수 있다.

## 권장 패턴

coroutine `Semaphore`를 사용하고 item lifecycle 전체 동안 permit을 유지한다.

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

limiter는 worker/service lifecycle에 한 번 만들고 batch call 사이에서 재사용한다.

```kotlin
private val workLimiter = CoroutineConcurrencyLimiter(concurrency)

suspend fun processDueJobs(batchSize: Int) {
    val claimed = claim(batchSize)
    workLimiter.process(claimed) { job ->
        process(job)
    }
}
```

blocking JDBC나 jOOQ work는 project의 transaction executor 또는 virtual-thread-backed dispatcher 안에 둔다. semaphore는 logical job flow를 제한하는 도구이지 DB transaction execution policy를 대체하지 않는다.

## 공통 원칙

Dispatcher parallelism은 thread execution limit이고, semaphore는 in-flight workflow limit이다. suspended coroutine은 dispatcher capacity를 비우지만 worker concurrency에서는 여전히 하나의 진행 중 job으로 세야 하는 경우가 많다. limiter state 자체가 semaphore이므로 concurrency budget과 같은 lifecycle에 둔다.

## 점검 방법

`limitedParallelism`을 쓰는데도 test에서 `maxInFlight`가 configured concurrency를 넘는지 확인한다. claimed row마다 coroutine을 launch하고 dispatcher 선택만으로 downstream API pressure를 제한하려는 worker code를 찾는다. helper가 batch invocation마다 `Semaphore(n)`을 만드는지도 확인한다.

## 검증 방법

suspending call 전에 `inFlight` counter를 증가시키고, delay 또는 wait 후 감소시키며, 관측된 최대 in-flight count가 configured concurrency와 같은지 assert하는 test를 둔다. job 수는 limit보다 많아야 한다. 모든 coroutine이 job body에 한 번에 들어가면 test가 실패해야 한다.
