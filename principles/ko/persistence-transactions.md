# 영속성, 트랜잭션, 동시성 경계

## 원칙

DB schema, transaction, lease, worker concurrency는 각자 다른 계약이다. schema는 generated reference로 다루고, DB transaction은 thread-bound 특성을 존중하며, business retry와 scheduler retry는 분리한다.

## 적용 기준

- jOOQ codegen reference를 schema 계약으로 사용한다.
- raw SQL은 DSL로 표현할 수 없는 경우에만 사용한다.
- DB 읽기 묶음은 하나의 transaction unit으로 유지하고, 독립적인 non-DB I/O와만 병렬화한다.
- worker in-flight 제한은 dispatcher parallelism이 아니라 semaphore 같은 workflow-level limiter로 둔다.
- polling attempt, callback delivery, retry schedule은 scheduler metadata가 아니라 domain state로 저장한다.

## 피해야 할 신호

- `DSL.table(DSL.name(...))`, string column name, raw `Record` mapper가 반복된다.
- transaction callback 안에서 여러 DB select를 `async`로 나눈다.
- response mismatch 같은 business state를 scheduler exception retry로 표현한다.
- suspending worker가 dispatcher 제한만 믿고 외부 API에 과도한 in-flight 작업을 만든다.

## 검증 방법

- compile, static analysis, persistence integration test로 generated reference 사용을 확인한다.
- lease claim은 PostgreSQL/Testcontainers concurrency test로 중복 claim이 없는지 확인한다.
- coroutine worker는 최대 in-flight counter test로 제한이 지켜지는지 검증한다.
- polling worker는 crash, lease expiry, retry, callback idempotency를 각각 검증한다.
