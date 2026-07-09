---
title: Prefer jOOQ DSL for database lease claims
tags: [springboot-kotlin, jooq, persistence, locking]
scope: cross-project
status: draft
---

# Prefer jOOQ DSL for database lease claims

## Context
Use this when a Spring Boot Kotlin persistence adapter claims due work from a
database table with row-level locking, lease tokens, bounded batch size, and a
status transition, especially on PostgreSQL with jOOQ generated table
references.

## Wrong Direction
Do not keep a plain SQL string just because the query uses
`FOR UPDATE SKIP LOCKED`, `UPDATE`, or `RETURNING`. Raw SQL duplicates table and
column names that already exist in generated jOOQ references, hides drift until
runtime, and often requires manual bind casts that the type-safe DSL can avoid.

## Correct Pattern
First check whether the project jOOQ version exposes the needed lock and
returning APIs. For a simple claim, use generated references end to end:

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

Keep the operation inside the existing transaction boundary and map returned
generated records back to domain models at the adapter boundary.

## Reusable Insight
Many PostgreSQL lease-claim queries can be expressed as
`UPDATE ... WHERE id IN (SELECT ... FOR UPDATE SKIP LOCKED) RETURNING ...` with
jOOQ DSL. This preserves atomic claim behavior while keeping schema references
typed and reviewable.

## Detection
Search persistence adapters for `dsl.fetch("""`, `WITH picked`, table names in
raw strings, or `RETURNING <alias>.*` in claim methods. Before accepting raw SQL,
verify that jOOQ lacks a required construct rather than assuming DSL cannot
represent the query.

## Verification
Add a focused test or code review check proving the claim method uses jOOQ DSL
lock APIs such as `forUpdate()`, `skipLocked()`, and `returning()`. Run compile,
static analysis, and a PostgreSQL/Testcontainers concurrency test when Docker is
available to prove workers still claim each due row at most once.
