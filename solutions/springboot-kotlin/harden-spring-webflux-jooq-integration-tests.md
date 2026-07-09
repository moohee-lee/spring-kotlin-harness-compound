---
title: Harden Spring WebFlux jOOQ integration tests
tags: [springboot-kotlin, webflux, jooq, testing]
scope: cross-project
status: draft
---

# Harden Spring WebFlux jOOQ integration tests

## Context
Use this when a Spring Boot Kotlin service combines WebFlux functional routes,
jOOQ persistence, raw SQL for PostgreSQL-specific locking, and Spring Boot
integration tests that reuse application contexts.

## Wrong Direction
Do not assume MVC-style API version registration automatically applies to
functional routes. Do not rely on plain jOOQ SQL bind inference for PostgreSQL
types such as `TIMESTAMP WITH TIME ZONE` or `UUID` inside complex
`UPDATE ... RETURNING` statements. Do not make test schemas non-idempotent when
H2 named in-memory databases use `DB_CLOSE_DELAY=-1` and multiple Spring test
contexts run the same `INIT=RUNSCRIPT`.

## Correct Pattern
For functional WebFlux routes on Spring versions that use the new API
versioning support, register supported versions explicitly in the API version
config, then verify the versioned path with a router-level integration test.

When raw jOOQ SQL must use PostgreSQL-only constructs such as `FOR UPDATE SKIP
LOCKED` plus `UPDATE ... RETURNING`, cast bind placeholders in SQL for values
where the driver or jOOQ plain SQL path may otherwise send an ambiguous string:
`CAST(? AS TIMESTAMP WITH TIME ZONE)` and `CAST(? AS UUID)` are often clearer
than debugging runtime operator/type errors later.

Make shared test schema scripts idempotent with `CREATE TABLE IF NOT EXISTS`
and `CREATE INDEX IF NOT EXISTS`, or use unique database names per test
context. This is especially important when H2 named databases are kept alive for
the JVM and Spring loads the same schema script more than once.

## Reusable Insight
Integration failures around routing, SQL types, and schema bootstrapping often
surface only after the production slices are wired together. Harden them with
small focused integration tests before adding end-to-end tests, so the final
test explains business behavior rather than framework configuration surprises.

## Detection
Look for functional routes under versioned paths that return 400 before the
handler runs, PostgreSQL errors that compare timestamp columns to varchar bind
values, or H2 startup failures that say a table or index already exists during a
second Spring test context.

## Verification
Run a WebTestClient test against the actual versioned route, a PostgreSQL
Testcontainers test around the jOOQ raw SQL path, and at least two Spring
contexts that load the H2 schema script. The verification should fail at the
smallest layer when the API version list, SQL casts, or schema idempotency is
removed.
