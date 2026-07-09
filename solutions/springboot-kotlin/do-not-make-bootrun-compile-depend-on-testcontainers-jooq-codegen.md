---
title: bootRun compile path가 Testcontainers jOOQ codegen에 의존하지 않게 한다
tags: [springboot-kotlin, jooq, testcontainers, gradle, bootrun]
scope: cross-project
status: active
principles: [build-tooling, persistence-transactions]
---

# bootRun compile path가 Testcontainers jOOQ codegen에 의존하지 않게 한다

> 관련 공통 원칙: [빌드와 도구 체인은 런타임을 오염시키지 않는다](../../principles/ko/build-tooling.md), [영속성, 트랜잭션, 동시성 경계](../../principles/ko/persistence-transactions.md)

## 적용 시점

Spring Boot Kotlin project가 `org.testcontainers.jdbc.ContainerDatabaseDriver`로 jOOQ code generation을 구성해서 disposable PostgreSQL container에서 schema generation을 수행할 때 적용한다.

## 피해야 할 방향

다음처럼 `compileKotlin`이 항상 `jooqCodegen`에 의존하게 만들면 안 된다.

```kotlin
tasks.named("compileKotlin") {
    dependsOn(tasks.named("jooqCodegen"))
}
```

이렇게 하면 `bootRun`을 포함한 모든 compile path가 jOOQ code generation을 시작한다. codegen이 Testcontainers JDBC를 쓰면 애플리케이션 실행 자체는 Testcontainers가 필요 없는데도 local app startup에서 Ryuk connection error가 나거나 Docker가 필수 조건이 된다.

## 권장 패턴

`jooqCodegen`은 명시적인 schema-generation task로 유지한다. main source가 매 build마다 fresh generated class를 반드시 요구하지 않는다면 `compileKotlin`이나 `bootRun`에 무조건 연결하지 않는다. generated class가 필요하면 surprise-free generation workflow, committed/generated source 전략, 또는 분석 task에 한정된 dependency처럼 의도를 드러내는 방식을 선택한다.

## 공통 원칙

Build-time schema generation과 runtime app startup은 operational dependency가 다르다. 편한 compile dependency 하나가 Docker/Testcontainers를 개발자 boot path로 끌어올 수 있다.

## 점검 방법

```bash
./gradlew bootRun --dry-run
```

출력에서 `:jooqCodegen`이 `:bootRun` 앞에 나타나는지 확인한다. 나타난다면 jOOQ JDBC driver가 Testcontainers 기반인지 함께 확인한다.

## 검증 방법

compile-time dependency를 제거한 뒤 다시 `./gradlew bootRun --dry-run`을 실행해 `:jooqCodegen`이 없는지 확인한다. 이후 `./gradlew test --rerun-tasks`를 실행해 일반 compile/test path는 여전히 통과하는지 검증한다.
