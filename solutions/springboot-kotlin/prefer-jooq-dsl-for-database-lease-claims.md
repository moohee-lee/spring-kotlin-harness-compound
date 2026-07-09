---
title: DB lease claim에는 가능하면 jOOQ DSL을 사용한다
tags: [springboot-kotlin, jooq, persistence, locking]
scope: cross-project
status: active
principles: [persistence-transactions]
---

# DB lease claim에는 가능하면 jOOQ DSL을 사용한다

> 관련 공통 원칙: [영속성, 트랜잭션, 동시성 경계](../../principles/ko/persistence-transactions.md)

## 적용 시점

Spring Boot Kotlin persistence adapter가 database table에서 due work를 claim하고, row-level locking, lease token, bounded batch size, status transition을 함께 다룰 때 적용한다. PostgreSQL과 jOOQ generated table reference를 쓰는 경우가 대표적이다.

## 피해야 할 방향

`FOR UPDATE SKIP LOCKED`, `UPDATE`, `RETURNING`이 들어간다는 이유만으로 plain SQL string을 유지하면 안 된다. raw SQL은 generated jOOQ reference에 이미 있는 table/column name을 다시 쓰게 만들고, drift를 runtime까지 숨기며, type-safe DSL이 피할 수 있는 manual bind cast를 요구하는 경우가 많다.

## 권장 패턴

먼저 project jOOQ version이 필요한 lock/returning API를 제공하는지 확인한다. 단순 claim은 generated reference로 끝까지 표현한다.

```kotlin
dsl.update(JOB)
    .set(JOB.STATUS, Processing.value)
    .set(JOB.LOCK_TOKEN, lockToken)
    .set(JOB.ATTEMPT_COUNT, JOB.ATTEMPT_COUNT.plus(1))
    .where(
        JOB.ID.`in`(
            dsl.select(JOB.ID)
                .from(JOB)
                .where(JOB.STATUS.`in`(Ready.value, Waiting.value))
                .and(JOB.NEXT_RUN_AT.le(now))
                .and(JOB.LOCKED_UNTIL.isNull().or(JOB.LOCKED_UNTIL.lt(now)))
                .orderBy(JOB.NEXT_RUN_AT.asc())
                .limit(batchSize)
                .forUpdate()
                .skipLocked(),
        ),
    )
    .returning()
    .fetch()
```

operation은 기존 transaction boundary 안에 두고 returned generated record를 adapter boundary에서 domain model로 mapping한다.

## 공통 원칙

많은 PostgreSQL lease-claim query는 `UPDATE ... WHERE id IN (SELECT ... FOR UPDATE SKIP LOCKED) RETURNING ...` 형태로 jOOQ DSL로 표현할 수 있다. atomic claim behavior를 유지하면서 schema reference를 typed/reviewable하게 보존한다.

## 점검 방법

persistence adapter에서 `dsl.fetch("""`, `WITH picked`, raw string 안 table name, `RETURNING <alias>.*`를 검색한다. raw SQL을 받아들이기 전에 jOOQ가 필요한 construct를 정말 제공하지 않는지 확인한다.

## 검증 방법

claim method가 `forUpdate()`, `skipLocked()`, `returning()` 같은 jOOQ DSL lock API를 쓰는지 focused test나 code review check로 확인한다. Docker가 가능하면 compile, static analysis, PostgreSQL/Testcontainers concurrency test로 여러 worker가 due row를 중복 claim하지 않는지 검증한다.
