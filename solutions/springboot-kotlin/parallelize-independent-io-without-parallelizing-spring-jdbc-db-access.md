---
title: 독립 I/O만 병렬화하고 Spring JDBC DB 접근은 트랜잭션 단위로 유지한다
tags: [springboot-kotlin, coroutines, transactions, jdbc]
scope: cross-project
status: active
principles: [persistence-transactions]
---

# 독립 I/O만 병렬화하고 Spring JDBC DB 접근은 트랜잭션 단위로 유지한다

> 관련 공통 원칙: [영속성, 트랜잭션, 동시성 경계](../../principles/ko/persistence-transactions.md)

## 적용 시점

Spring Boot Kotlin service가 blocking JDBC/jOOQ read와 Feign/WebClient 같은 독립 non-DB I/O call을 함께 수행할 때 적용한다. Spring JDBC transaction state는 thread-bound이므로 하나의 logical read flow에 속한 DB work는 하나의 transaction callback 안에 남아야 한다.

## 피해야 할 방향

별도 DB select를 각각 `async` block에 넣거나 `TransactionTemplate` callback 안에서 coroutine fan-out utility를 호출하면 안 된다. DB access가 thread와 transaction을 넘나들 수 있고, read consistency와 transaction routing을 추론하기 어려워진다.

## 권장 패턴

하나의 logical snapshot을 공유할 수 있는 DB read는 단일 transaction-port call로 묶고, 그 전체 DB-read unit만 독립 non-DB I/O와 병렬 실행한다.

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

## 공통 원칙

병렬화 경계는 DB transaction 내부가 아니라 complete DB transaction 바깥에 있어야 한다. 독립적인 non-DB I/O와 DB I/O는 overlap할 수 있지만, DB access 자체는 transaction 안에서 순차적으로 유지한다.

## 점검 방법

하나의 flow에서 `transactionalPort.executeReadOnly`를 여러 번 호출하거나 여러 `async` block에서 DB port를 호출하는 service를 찾는다. DB read가 하나의 logical snapshot에 속한다면 coroutine parallelism을 추가하기 전에 하나의 transaction call로 합친다.

## 검증 방법

transaction port test double을 만들어 `executeReadOnly` 호출 횟수를 센다. non-DB call과 DB transaction unit에 같은 delay를 넣은 virtual-time coroutine test를 작성한다. 병렬 실행은 delay 한 번의 시간으로 끝나야 하지만 transaction count는 하나여야 한다.
