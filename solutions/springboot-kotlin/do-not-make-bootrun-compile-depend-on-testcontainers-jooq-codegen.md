---
title: Do not make bootRun compile depend on Testcontainers jOOQ codegen
tags: [springboot-kotlin, jooq, testcontainers, gradle, bootrun]
scope: cross-project
status: active
---

# Do not make bootRun compile depend on Testcontainers jOOQ codegen

## Context

Spring Boot Kotlin projects may configure jOOQ code generation with `org.testcontainers.jdbc.ContainerDatabaseDriver` so schema generation can run against a disposable PostgreSQL container.

## Wrong Direction

Adding `tasks.named("compileKotlin") { dependsOn(tasks.named("jooqCodegen")) }` makes every compile path, including `bootRun`, start jOOQ code generation. If code generation uses Testcontainers JDBC, ordinary local app startup can emit Ryuk connection errors or require Docker even when the application itself does not need Testcontainers.

## Correct Pattern

Keep `jooqCodegen` available as an explicit schema-generation task, but do not wire it unconditionally into `compileKotlin` or `bootRun` unless main source code truly requires freshly generated classes on every build. If generated classes are needed, prefer a deliberate generation workflow or committed/generated source strategy that does not surprise app startup.

## Reusable Insight

Build-time schema generation and runtime app startup have different operational dependencies. A convenient compile dependency can accidentally move Docker/Testcontainers into the developer boot path.

## Detection

Run `./gradlew bootRun --dry-run` and check whether `:jooqCodegen` appears before `:bootRun`. If yes, inspect whether the jOOQ JDBC driver is Testcontainers-based.

## Verification

After removing the compile-time dependency, run `./gradlew bootRun --dry-run` and confirm `:jooqCodegen` is absent. Run `./gradlew test --rerun-tasks` to ensure normal compile/test paths still succeed.
