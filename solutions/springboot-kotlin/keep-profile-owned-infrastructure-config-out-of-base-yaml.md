---
title: Keep profile-owned infrastructure config out of base YAML
tags: [springboot-kotlin, configuration, profiles, secrets]
scope: cross-project
status: active
---

# Keep profile-owned infrastructure config out of base YAML

## Context

Spring Boot services often have base `application.yaml` plus profile-specific files such as `application-dev.yaml`, `application-local.yaml`, and `application-test.yaml`. Infrastructure endpoints and credentials for datasource, Kafka, Vault, and similar adapters usually differ by profile.

## Wrong Direction

Keeping datasource/Kafka endpoints in base `application.yaml` with placeholders like `${KAFKA_BOOTSTRAP_SERVERS:localhost:19092}` makes base configuration environment-specific and lets shell environment variables silently override profile intent. It also blurs which profile owns which infrastructure contract.

## Correct Pattern

Keep base `application.yaml` limited to truly shared application settings. Put datasource, Kafka client properties, Kafka topics, Vault imports, and credential placeholders in `application-{profile}.yaml`. Use literal profile values for non-secret endpoints and modes, and use Vault-prefixed placeholders only for secrets.

When a Kubernetes ConfigMap is the profile-specific runtime configuration
source, generate or curate it from the owning profile file. Put non-secret
runtime endpoints such as datasource JDBC URLs and Kafka bootstrap servers
directly in the ConfigMap `application.yml`; reserve Deployment environment
variables for pod metadata, bootstrap toggles, or values that are intentionally
outside the application profile contract.

## Reusable Insight

Profile files should be the source of truth for infrastructure wiring. Environment variables are useful for secrets or deployment parameters that are explicitly part of the contract, but `${ENV:default}` fallbacks in shared YAML make review and runtime behavior harder to reason about.

## Detection

Search base `application.yaml` for `spring.datasource`, `spring.kafka`, `app.kafka`, and `${...:...}` fallback placeholders. If found, ask whether those values are truly profile-independent; most datasource and Kafka values are not.
For Kubernetes deployments, also search Helm values and rendered Deployments
for app-owned endpoint names such as `DB_JDBC_URL` or
`KAFKA_BOOTSTRAP_SERVERS`; if the same data belongs to the ConfigMap-backed
profile, move it into that ConfigMap instead of injecting it as container env.

## Verification

Add configuration tests that load base and profile YAML separately. Assert base YAML lacks profile-owned infrastructure keys, and assert each profile file carries the expected literal datasource/Kafka values plus Vault-prefixed secret placeholders. Run the full Spring context tests for each active test profile.
For ConfigMap-backed profiles, parse the raw ConfigMap manifest and its
embedded `application.yml`, then assert non-secret endpoints are present there
and absent from environment-specific Helm values.
