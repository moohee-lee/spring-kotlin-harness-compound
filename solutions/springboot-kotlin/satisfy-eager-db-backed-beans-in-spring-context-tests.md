---
title: Satisfy eager DB backed beans in Spring context tests
tags: [springboot-kotlin, testing, bean-lifecycle]
scope: cross-project
status: draft
---

# Satisfy eager DB backed beans in Spring context tests

## Context

Use this when a Spring Boot bean is intentionally changed from lazy/runtime loading
to eager constructor-time loading from a database-backed port. This commonly applies
to application-scoped holders for required system properties, tenant metadata, or
region identifiers.

## Wrong Direction

Only update the unit tests around the holder or service and assume disabled runners
make full Spring context tests irrelevant. A disabled `ApplicationRunner` does not
stop ordinary singleton services and their constructor dependencies from being
created during context startup.

## Correct Pattern

Keep the eager-loading production contract explicit: inject the output port into
the holder and fail bean creation when the required value is missing. For Spring
context tests, satisfy that required dependency deliberately with one of these
test-owned inputs:

- a `@TestConfiguration` primary port fake when the test is not about persistence,
- test database seed data when the test must exercise the real persistence adapter.

Choose the smallest option that preserves the test's purpose.

## Reusable Insight

Moving a database read into bean construction expands the blast radius from the
runtime use case to every Spring context that creates the bean. Test profiles,
disabled startup runners, and mocked external clients do not remove that
constructor-time requirement.

## Detection

After an eager DB-backed bean change, run at least one full `@SpringBootTest`
context test. Look for `BeanCreationException`, `UnsatisfiedDependencyException`,
or the holder's missing-configuration exception during context startup.

## Verification

Add or update a focused holder test for successful eager loading and missing-value
failure. Then run the affected service tests plus a Spring context test that would
create the bean. A full test suite run should no longer include context startup
failures from the eager bean.
