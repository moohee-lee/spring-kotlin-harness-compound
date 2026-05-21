---
title: Split util-only libraries from Spring Boot starters
tags: [springboot-kotlin, harness, dependencies, autoconfiguration]
scope: cross-project
status: draft
---

# Split util-only libraries from Spring Boot starters

## Context
Spring Boot services sometimes need one plain utility package from a shared company library, while that shared library also ships auto-configuration metadata, logging configuration, message bundles, property sources, web interceptors, data source configuration, or other runtime side effects.

## Wrong Direction
Adding the whole shared library as a normal runtime dependency and trying to "only import one package" is not a reliable isolation boundary. Gradle dependency excludes remove transitive modules, not selected classes or resources inside the same jar. Once the jar is on the runtime classpath, Spring Boot can discover `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`, and logging can discover classpath logging resources such as `logback-spring.xml`.

## Correct Pattern
Publish separate artifacts:

- A plain `java-library` or Kotlin library artifact for utility code, models, constants, and exceptions. It should not contain Spring Boot auto-configuration imports, application logging files, or environment-specific resources.
- A separate starter/autoconfigure artifact that depends on the utility artifact and owns Spring Boot auto-configuration, logging integration, web adapters, message sources, and infrastructure beans.

Consumers that only need utility behavior depend on the util artifact. Consumers that want the full platform behavior depend on the starter.

## Reusable Insight
Classpath is the runtime boundary, not Java package import syntax. If a jar contains both utilities and Boot integration metadata, using one class from it still brings the metadata and resources along.

## Detection
Before adding a shared dependency for a utility, inspect the jar or source for `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`, `META-INF/spring.factories`, `logback.xml`, `logback-spring.xml`, message/property resources, and broad `@AutoConfiguration` classes. If present, treat it as a starter-style dependency, not a utility-only dependency.

## Verification
In the consumer service, run an `ApplicationContextRunner` or focused boot startup test with the dependency present and assert that unwanted beans/auto-configurations are absent. Also verify logging initialization uses the application's own logging config. For the long-term fix, inspect the util artifact jar and confirm it contains only the intended utility packages/resources and no Boot metadata.
