---
title: jOOQ insert와 select mapping은 generated table record로 모은다
tags: [springboot-kotlin, jooq, persistence]
scope: cross-project
status: active
principles: [persistence-transactions]
---

# jOOQ insert와 select mapping은 generated table record로 모은다

> 관련 공통 원칙: [영속성, 트랜잭션, 동시성 경계](../../principles/ko/persistence-transactions.md)

## 적용 시점

Spring Boot Kotlin persistence adapter가 jOOQ generated table record를 사용하고, database row와 domain model 사이를 mapping할 때 적용한다.

## 피해야 할 방향

insert value를 긴 `.set(TABLE.FIELD, value)` chain으로 흩뿌리고 mapper function이 raw `org.jooq.Record`를 받게 두면 table-column 지식이 query code 전반에 퍼진다. select와 insert path가 서로 다른 mapping shape를 쓰는 문제도 생긴다.

## 권장 패턴

adapter-local mapper function을 만들어 domain model과 generated table record 사이를 변환한다. 예를 들어 `Domain.toRecord(): DomainRecord`, `DomainRecord.toEntity(): Domain`을 둔다.

insert에서는 generated record를 먼저 만들고 jOOQ에 넘긴다.

```kotlin
dsl.insertInto(TABLE)
    .set(domain.toRecord())
    .execute()
```

select-like result는 query boundary에서 generated table record로 변환한 뒤 domain으로 mapping한다.

```kotlin
dsl.fetch(sql, args)
    .into(TABLE)
    .map { it.toEntity() }
```

guarded condition 아래 일부 column만 update하는 update query는 의도적으로 field-based로 유지할 수 있다.

## 공통 원칙

Generated jOOQ record는 persistence adapter 내부 boundary type이다. domain service에는 domain model만 보여야 하지만 adapter 내부에서는 generated record를 일관되게 사용해야 column mapping이 한곳에 모이고 review하기 쉽다.

## 점검 방법

persistence adapter에서 `import org.jooq.Record`, raw `Record`를 받는 mapper function, 모든 table column을 반복하는 insert chain을 찾는다. 이런 신호가 있으면 generated record mapping을 중앙화할 대상이다.

## 검증 방법

Docker/Testcontainers나 target database를 사용할 수 있으면 일반 persistence integration test를 실행한다. review-driven refactor라면 lightweight structural test로 adapter가 domain-to-generated-record mapper를 노출하고 private raw `Record` mapper entry point를 더 이상 갖지 않는지 확인할 수도 있다.
