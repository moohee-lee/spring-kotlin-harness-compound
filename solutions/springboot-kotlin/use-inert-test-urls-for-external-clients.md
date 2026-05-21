---
title: Use inert test URLs for external clients
tags: [springboot-kotlin, configuration, testing, http-client]
scope: cross-project
status: draft
---

# Use inert test URLs for external clients

## Context

Spring Boot services often configure Feign, WebClient, RestClient, Kafka schema registry, or other external adapter endpoints through profile YAML. Production-like profiles may need real internal URLs, while the test profile only needs enough configuration for context startup and focused unit tests.

## Wrong Direction

Copying a realistic dev, stage, or test environment hostname into `application-test.yaml` makes accidental integration calls look legitimate. A future context test, smoke test, or mistakenly wired client can reach an internal service instead of failing fast inside the local test process.

## Correct Pattern

Keep real external endpoints in the owning runtime profile, such as `application-dev.yaml` or deployment-managed configuration. In `application-test.yaml`, use an inert local URL such as `http://localhost` unless the test explicitly starts a local stub server and points the property at that server.

## Reusable Insight

Test configuration should satisfy binding and bean creation without implying access to real infrastructure. A local placeholder URL makes accidental calls noisy and contained, while still preserving the same property path that production code binds.

## Detection

Search test resources for hostnames that look like real shared environments, such as `dev`, `stage`, `internal`, cloud domains, or service-discovery names. If a test profile uses one, ask whether any test intentionally calls that endpoint; if not, replace it with an inert local URL.

## Verification

Add configuration tests that load base and profile YAML separately. Assert that real endpoint values live only in runtime profiles, and that the test profile uses a local or stub-owned endpoint. Run the full Spring context tests to prove the placeholder still satisfies bean creation.
