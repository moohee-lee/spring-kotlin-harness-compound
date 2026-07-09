---
title: map jooq insert and select through generated table records
tags: [springboot-kotlin, jooq, persistence]
scope: cross-project
status: draft
---

# map jooq insert and select through generated table records

## Context

Use this when a Spring Boot Kotlin persistence adapter uses jOOQ generated table records and maps between database rows and domain models.

## Wrong Direction

Scattering insert values with long `.set(TABLE.FIELD, value)` chains and accepting raw `org.jooq.Record` in mapper functions spreads table-column knowledge across query code. It also makes select and insert paths use different mapping shapes.

## Correct Pattern

Create adapter-local mapper functions between the domain model and the generated table record, for example `Domain.toRecord(): DomainRecord` and `DomainRecord.toEntity(): Domain`.

For inserts, build the generated record first and pass it to jOOQ:

```kotlin
dsl.insertInto(TABLE)
    .set(domain.toRecord())
    .execute()
```

For select-like results, convert raw query results into the generated table record at the query boundary, then map to domain:

```kotlin
dsl.fetch(sql, args)
    .into(TABLE)
    .map { it.toEntity() }
```

Keep update queries field-based when they intentionally update only a subset of columns under guarded conditions.

## Reusable Insight

Generated jOOQ records are the persistence adapter boundary type. Domain services should still see only domain models, but inside the adapter, using the generated record consistently keeps column mapping centralized and easier to review.

## Detection

Look for persistence adapters with `import org.jooq.Record`, mapper functions that accept raw `Record`, or insert chains that repeat every table column. Those are signs that generated record mapping should be centralized.

## Verification

Use normal persistence integration tests when Docker/Testcontainers or the target database is available. For review-driven refactors, a lightweight structural test can also assert that the adapter exposes a domain-to-generated-record mapper and no longer has private raw `Record` mapper entry points.
