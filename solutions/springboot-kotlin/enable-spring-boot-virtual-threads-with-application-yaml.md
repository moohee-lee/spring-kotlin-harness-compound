---
title: enable spring boot virtual threads with application yaml
tags: [springboot-kotlin, spring-boot, virtual-threads]
scope: cross-project
status: draft
---

# enable spring boot virtual threads with application yaml

## Context

Use this when a Spring Boot Kotlin service runs on Java 21+ and should let Spring Boot use virtual threads for its auto-configured application task execution.

## Wrong Direction

Only creating ad-hoc `Executors.newVirtualThreadPerTaskExecutor()` instances in a few adapters does not enable Spring Boot's global virtual-thread mode. It leaves Boot-managed task execution on the default platform-thread configuration.

## Correct Pattern

Set the Boot property in `application.yaml`:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

Keep any explicitly scoped virtual-thread dispatchers that are required for blocking JDBC or other thread-bound integrations. The Boot property complements those targeted executors.

## Reusable Insight

For Spring Boot 3.2+ and 4.x, `spring.threads.virtual.enabled` is the framework-level switch. In WebFlux applications this does not turn Netty event loops into virtual threads; it configures Boot-managed task execution where applicable while nonblocking event loops remain event loops.

## Detection

Search `application.yaml` and profile-specific config for `spring.threads.virtual.enabled`. Also search production code for custom virtual-thread executors to understand which blocking work has separate, explicit execution boundaries.

## Verification

Add a lightweight YAML binding test that asserts `spring.threads.virtual.enabled` is `true`, and run a Spring context load test to confirm the application starts with the setting. Full integration tests may still depend on external services such as Docker/Testcontainers.
