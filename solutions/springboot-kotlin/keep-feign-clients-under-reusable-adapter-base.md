---
title: Keep Feign clients under reusable adapter base
tags: [springboot-kotlin, feign, architecture, configuration]
scope: cross-project
status: draft
---

# Keep Feign clients under reusable adapter base

## Context

Spring Boot Kotlin services often start with one Feign client for one gateway, then add more clients over time. Early package and bean names tend to become the default pattern future contributors copy.

## Wrong Direction

Scanning only one client package such as `adapter.output.feign.identity`, or naming shared infrastructure `identityFeignCoroutineDispatcher`, makes future clients add duplicate scan annotations, duplicate dispatcher beans, or domain-specific configuration that is not actually domain-specific.

## Correct Pattern

Use a reusable base package such as `adapter.output.feign` for `@EnableFeignClients`. Keep that scan annotation on a shared Feign infrastructure configuration such as `FeignClientConfiguration`, not on the application class. Put clients under `adapter.output.feign.{domain}` and shared Feign infrastructure under `adapter.output.feign.config` or `adapter.output.feign.support`. Only create client-specific configuration when timeout, retry, pool, circuit breaker, or protocol policy genuinely differs.

## Reusable Insight

For signed Feign calls, keep a reusable opt-in configuration such as `SignatureFeignConfiguration` in the shared Feign config package. Do not register that configuration or its interceptor as a global `@Component`/`@Configuration`; attach it only from Feign clients that need signatures with `@FeignClient(configuration = [...])`.

Avoid reflecting Feign interface annotations to reconstruct runtime URLs. `@FeignClient(url = "\${...}")` is a placeholder and the effective target is resolved during the Spring Cloud OpenFeign lifecycle. Derive the HTTP method and final target URL inside the interceptor from Feign's resolved `RequestTemplate`.

When a pure signing function already exists, keep the credential `ConfigurationProperties` near the signed HTTP adapter, for example as `SignatureSecretProperties` under the Feign config package. Inject it into the opt-in Feign configuration and call the pure function directly. Do not add a Spring service whose only behavior is copying properties into the same function; it creates another bean and another test surface without adding policy.

## Detection

Search for `@EnableFeignClients(basePackages = [` values that end at one domain package, Feign scan annotations left on the application class, bean names that include a gateway name for otherwise generic infrastructure, signed-client configurations duplicated under domain packages, signature interceptors registered with stereotype annotations, and services that only delegate configuration properties into a pure signer.

## Verification

Add focused tests that assert the shared Feign configuration scans the common Feign base package, the application class does not own Feign scanning, the adapter runs blocking Feign calls on the shared virtual-thread dispatcher, and signature interceptors are opt-in through the Feign client's `configuration` rather than global `@Component` beans. For signed clients, test that the interceptor derives method and URL from Feign's resolved `RequestTemplate`. Run detekt and the full test suite after package moves.
