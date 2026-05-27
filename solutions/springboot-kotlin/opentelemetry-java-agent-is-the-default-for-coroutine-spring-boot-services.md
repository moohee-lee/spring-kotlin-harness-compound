---
title: OpenTelemetry Java agent is the default for coroutine Spring Boot services
tags: [springboot-kotlin, opentelemetry, coroutines, observability]
scope: cross-project
status: draft
---

# OpenTelemetry Java agent is the default for coroutine Spring Boot services

## Context
Spring Boot Kotlin services commonly combine WebFlux/Reactor, `suspend`
application services, `withContext` on custom coroutine dispatchers, JDBC/jOOQ,
Kafka, and Logback. Teams may ask whether OpenTelemetry requires app code
changes because coroutine execution can move across threads.

## Wrong Direction
Do not start by adding the full OpenTelemetry SDK and custom span lifecycle
management to the application just because the service uses Kotlin coroutines.
Also do not keep custom MDC `traceId` logging as the only correlation key after
enabling the OpenTelemetry Java agent; the agent writes standard MDC keys such
as `trace_id` and `span_id`, while custom request filters may generate a
different trace id when no inbound `traceparent` exists.

## Correct Pattern
Prefer the OpenTelemetry Java agent for normal Spring Boot JVM deployments. It
provides broad automatic instrumentation, including context propagation for
Kotlin coroutines, Reactor, Java executors, WebFlux, Kafka, JDBC, HikariCP, and
Logback MDC. Add application OpenTelemetry dependencies only when the service
needs manual spans, custom attributes, or starter-based configuration instead
of the Java agent. For manual spans inside coroutine code, add the OpenTelemetry
API and Kotlin context extension and propagate `Context.current()` as a
coroutine context element around `withContext`, `async`, or launched work.

## Reusable Insight
Coroutine usage changes the context-propagation check, not the default
instrumentation choice. With the Java agent, most propagation is automatic; the
main project-specific work is Kubernetes agent injection/configuration and log
correlation cleanup.

## Detection
Look for custom `CoroutineDispatcher` beans, `withContext`, `async`, or
`runBlocking`; then check whether OpenTelemetry is being added through the Java
agent/operator or through app dependencies. Inspect `logback-spring.xml` for
custom `%X{traceId}` patterns that should be aligned with OTel `%X{trace_id}`
and `%X{span_id}` once agent MDC injection is enabled.

## Verification
Run the app with the Java agent and a local or in-cluster Collector. Exercise
HTTP, outbound HTTP/Feign, Kafka, and JDBC paths, then verify one trace links
server spans, child client/database/messaging spans, and log lines containing
the same `trace_id`. For coroutine paths, verify spans remain parented across
`withContext` dispatchers and `async` fan-out.
