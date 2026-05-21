---
title: Use jOOQ codegen references in persistence adapters
tags: [springboot-kotlin, jooq, codegen, persistence]
scope: cross-project
status: active
---

# Use jOOQ codegen references in persistence adapters

## Context

Use this when a Spring Boot Kotlin persistence adapter writes jOOQ queries against tables that are already available through generated `tables.references` classes.

## Wrong Direction

Do not duplicate schema knowledge with `DSL.table(DSL.name(...))`, `DSL.field(DSL.name(...))`, string column names, or `record.get("column", Type::class.java)` in normal adapters. That hides table/column drift until runtime and makes aliases, nullability, and selected field mapping harder to review.

## Correct Pattern

Import generated table references from `com.<project>.jooq.generated.tables.references`, alias generated table instances when SQL aliases are needed, and select/read generated `Field` objects end to end:

- `select(TABLE.COLUMN)` instead of `select(DSL.field(...))`
- `from(TABLE)` instead of `from(DSL.table(...))`
- Alias generated tables only when the SQL actually needs it, such as self-joins, joining the same table twice, or disambiguating an unavoidable naming conflict. Simple one-use joins should import the generated references directly without Kotlin import aliases or jOOQ table aliases.
- `record.get(TABLE.COLUMN)` or helper methods that accept `Field<T>` instead of string names

If generated sources are written under `build/` and main code imports them, clean builds need an explicit generation path such as `compileKotlin` and analysis tasks depending on `jooqCodegen`. For Testcontainers-backed codegen, check the existing bootRun/Docker tradeoff before wiring that dependency globally.

## Reusable Insight

jOOQ codegen is most useful when adapters treat generated references as the schema contract. Mixing generated schema and ad hoc string DSL creates two sources of truth.

## Detection

Search persistence adapters for `DSL.table`, `DSL.field`, `DSL.name`, `record.get("...")`, or duplicated table/column constants. Tests can assert schema-qualified generated table names in rendered SQL to make accidental raw table usage visible.

## Verification

Run `./gradlew clean test --tests '*<AdapterTest>'` to prove generated sources are available after clean, then run the project's normal static analysis and test command. For Testcontainers-backed codegen, also run `./gradlew bootRun --dry-run` and deliberately accept or redesign any `:jooqCodegen` dependency in the app startup path.
