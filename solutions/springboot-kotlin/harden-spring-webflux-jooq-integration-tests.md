---
title: Spring WebFlux jOOQ integration test를 작은 실패 지점으로 단단하게 만든다
tags: [springboot-kotlin, webflux, jooq, testing]
scope: cross-project
status: active
principles: [testing-contracts, persistence-transactions]
---

# Spring WebFlux jOOQ integration test를 작은 실패 지점으로 단단하게 만든다

> 관련 공통 원칙: [테스트는 계약을 검증해야 한다](../../principles/ko/testing-contracts.md), [영속성, 트랜잭션, 동시성 경계](../../principles/ko/persistence-transactions.md)

## 적용 시점

Spring Boot Kotlin service가 WebFlux functional route, jOOQ persistence, PostgreSQL-specific locking raw SQL, Spring Boot integration test context reuse를 함께 사용할 때 적용한다.

## 피해야 할 방향

MVC-style API version registration이 functional route에도 자동 적용된다고 가정하면 안 된다. `UPDATE ... RETURNING`이나 `FOR UPDATE SKIP LOCKED` 같은 PostgreSQL-specific SQL에서 plain jOOQ SQL bind inference가 `TIMESTAMP WITH TIME ZONE`, `UUID` 같은 type을 항상 정확히 처리한다고 믿는 것도 위험하다. H2 named in-memory database가 `DB_CLOSE_DELAY=-1`로 유지되고 여러 Spring test context가 같은 `INIT=RUNSCRIPT`를 실행한다면 non-idempotent schema도 피해야 한다.

## 권장 패턴

새 API versioning support를 쓰는 Spring version에서 functional WebFlux route를 사용한다면 supported version을 API version config에 명시적으로 등록하고, versioned path는 router-level integration test로 확인한다.

raw jOOQ SQL이 PostgreSQL-only construct를 써야 한다면 ambiguous string bind를 피하기 위해 필요한 placeholder를 SQL 안에서 cast한다.

```sql
CAST(? AS TIMESTAMP WITH TIME ZONE)
CAST(? AS UUID)
```

H2 shared test schema script는 `CREATE TABLE IF NOT EXISTS`, `CREATE INDEX IF NOT EXISTS`로 idempotent하게 만들거나 test context마다 unique database name을 사용한다.

## 공통 원칙

routing, SQL type, schema bootstrap 문제는 production slice를 함께 wiring한 뒤에야 드러나는 경우가 많다. end-to-end test가 business behavior를 설명하게 만들려면, 그 전에 작은 focused integration test로 framework configuration surprise를 제거해야 한다.

## 점검 방법

versioned functional route가 handler 실행 전 400을 반환하는지, PostgreSQL error가 timestamp column과 varchar bind value 비교를 말하는지, 두 번째 Spring test context에서 H2 table/index already exists가 나는지 확인한다.

## 검증 방법

actual versioned route에 대한 WebTestClient test, jOOQ raw SQL path에 대한 PostgreSQL Testcontainers test, H2 schema script를 load하는 두 개 이상의 Spring context test를 둔다. API version list, SQL cast, schema idempotency 중 하나를 제거하면 가장 작은 layer에서 실패해야 한다.
