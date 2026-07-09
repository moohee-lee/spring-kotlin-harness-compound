---
title: wrap kotlin uuid v7 generation behind a java uuid top level function
tags: [springboot-kotlin, kotlin, uuid-v7]
scope: cross-project
status: draft
---

# wrap kotlin uuid v7 generation behind a java uuid top level function

## Context

Use this in Spring Boot Kotlin services that store or expose `java.util.UUID` but want newly generated identifiers to be UUID v7 for better temporal ordering.

## Wrong Direction

Calling `UUID.randomUUID()` directly in services, workers, filters, or adapters scatters v4 generation across the project. Replacing each call with low-level bit manipulation or a third-party UUID library adds avoidable code and dependency surface when the Kotlin stdlib already provides UUID v7 generation.

## Correct Pattern

Create one project-local Kotlin top-level function, for example `uuidV7(): java.util.UUID`, in a common utility package. Implement it with `kotlin.uuid.Uuid.generateV7().toJavaUuid()` and keep `@OptIn(ExperimentalUuidApi::class)` inside that wrapper.

Call that function directly at simple ID creation sites. Keep injectable generator lambdas only where they remove real complexity, such as worker lease/token flows that need deterministic concurrency or retry tests.

## Reusable Insight

Kotlin's UUID API can provide v7 generation while the rest of a Spring/JPA/jOOQ/WebFlux codebase continues to use `java.util.UUID`. A tiny wrapper keeps experimental API annotations and conversion details out of domain/application code.

## Detection

Search production code for `UUID.randomUUID`, `UUID::randomUUID`, or direct UUID library calls. Generated IDs, lock tokens, trace IDs, outbox IDs, and callback idempotency IDs should go through the shared generator unless they intentionally need a different UUID version. Also review constructor parameters for default generator/clock lambdas that only forward to a simple function call; remove them unless tests or runtime configuration genuinely need the seam.

## Verification

Add tests that assert generated UUIDs have `version() == 7` and RFC 4122 variant `variant() == 2`. Also test the consuming defaults, not only the utility function, so future call sites cannot drift back to v4. For services that should stay simple, add a lightweight constructor-shape or wiring test only when review feedback specifically targets unnecessary injection.
