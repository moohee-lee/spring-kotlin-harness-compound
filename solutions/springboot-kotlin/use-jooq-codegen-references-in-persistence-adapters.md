---
title: jOOQ persistence adapter는 codegen reference를 사용한다
tags: [springboot-kotlin, jooq, codegen, persistence]
scope: cross-project
status: active
principles: [persistence-transactions]
---

# jOOQ persistence adapter는 codegen reference를 사용한다

> 관련 공통 원칙: [영속성, 트랜잭션, 동시성 경계](../../principles/ko/persistence-transactions.md)

## 적용 시점

Spring Boot Kotlin persistence adapter가 이미 generated `tables.references` class로 제공되는 table에 대해 jOOQ query를 작성할 때 적용한다.

## 피해야 할 방향

일반 adapter에서 `DSL.table(DSL.name(...))`, `DSL.field(DSL.name(...))`, string column name, `record.get("column", Type::class.java)`로 schema 지식을 중복하면 안 된다. 이런 코드는 table/column drift를 runtime까지 숨기고 alias, nullability, selected field mapping을 review하기 어렵게 만든다.

## 권장 패턴

`com.<project>.jooq.generated.tables.references`의 generated table reference를 import한다. SQL alias가 필요하면 generated table instance를 alias한다. select/read는 generated `Field` object로 끝까지 유지한다.

- `select(TABLE.COLUMN)`을 사용하고 `select(DSL.field(...))`를 피한다.
- `from(TABLE)`을 사용하고 `from(DSL.table(...))`를 피한다.
- self-join, 같은 table 두 번 join, 불가피한 naming conflict처럼 SQL상 alias가 필요한 경우에만 generated table을 alias한다. 단순 one-use join은 Kotlin import alias나 jOOQ table alias 없이 generated reference를 직접 import한다.
- `record.get(TABLE.COLUMN)` 또는 `Field<T>`를 받는 helper를 사용하고 string name을 피한다.

generated source가 `build/` 아래 만들어지고 main code가 import한다면 clean build에는 `compileKotlin`과 analysis task가 `jooqCodegen`을 거치는 명시적 generation path가 필요할 수 있다. Testcontainers-backed codegen이면 bootRun/Docker tradeoff를 확인한 뒤 global wiring 여부를 결정한다.

## 공통 원칙

jOOQ codegen의 가치는 adapter가 generated reference를 schema contract로 취급할 때 가장 크다. generated schema와 ad hoc string DSL을 섞으면 source of truth가 둘로 갈라진다.

## 점검 방법

persistence adapter에서 `DSL.table`, `DSL.field`, `DSL.name`, `record.get("...")`, duplicated table/column constant를 검색한다. rendered SQL의 schema-qualified generated table name을 assert하는 test로 accidental raw table usage를 드러낼 수도 있다.

## 검증 방법

```bash
./gradlew clean test --tests '*<AdapterTest>'
```

clean 이후 generated source가 사용 가능한지 확인한 뒤 project의 static analysis와 normal test command를 실행한다. Testcontainers-backed codegen이면 `./gradlew bootRun --dry-run`도 실행해 app startup path에 `:jooqCodegen` dependency를 받아들일지, redesign할지 명시적으로 결정한다.
