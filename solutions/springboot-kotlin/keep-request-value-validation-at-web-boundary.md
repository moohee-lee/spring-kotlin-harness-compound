---
title: keep request value validation at web boundary
tags: [springboot-kotlin, validation, hexagonal-architecture]
scope: cross-project
status: draft
---

# keep request value validation at web boundary

## Context

Use this when a Spring Boot Kotlin service follows hexagonal architecture and an inbound WebFlux handler binds request bodies, path variables, or query parameters into request DTOs before calling an application service.

## Wrong Direction

Duplicating basic request value validation in the application service with Kotlin `require` looks harmless, but it moves transport input concerns into the use case and leaks `IllegalArgumentException` for client-input failures. Examples include checking that request DTO numbers are positive after the handler already validated `@Min(1)`.

## Correct Pattern

Let the inbound adapter validate request value shape and primitive constraints before creating the command. Convert those failures into the project's request validation exception shape, including field source (`BODY`, `QUERY`, `PATH`, or `HEADER`) and field errors.

Keep the application service focused on use-case policy: whether the already-normalized command is allowed for that service operation, state transition, tenant, feature flag, or domain rule. Use domain/application exceptions for those failures, not transport validation exceptions.

## Reusable Insight

The same numeric or string constraint can look like a service invariant, but if it is about whether the HTTP request value is syntactically valid, it belongs at the adapter boundary. If it is about whether the service operation permits that valid value in the current business context, it belongs in the service/domain layer.

## Detection

Review service methods for `require`, `check`, or `IllegalArgumentException` around request DTO constraints that are already expressed as Bean Validation annotations on inbound request classes. Also check common validation helpers: they should emit the project's request validation exception rather than raw Bean Validation exceptions when used by handlers.

## Verification

Add one handler/router test showing invalid request values return a 400 validation response before the use case is called. Add one application service test or code review assertion showing the service does not repeat the primitive request-value checks. Add a common validation helper test if the project has a custom `awaitBodyValidated` or query binding helper.
