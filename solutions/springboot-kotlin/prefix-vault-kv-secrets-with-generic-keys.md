---
title: Prefix Vault KV secrets with generic keys
tags: [springboot-kotlin, vault, configuration, secrets]
scope: cross-project
status: active
---

# Prefix Vault KV secrets with generic keys

## Context

Spring Boot services using Spring Cloud Vault ConfigData can import multiple KV paths as property sources. Some Vault stores use generic key names such as `USER` and `PASSWORD`, while application configuration often needs distinct names for database, Kafka, API signature, and other credentials.

## Wrong Direction

Referencing generic Vault keys directly, such as `${USER}` or `${PASSWORD}`, makes configuration fragile. These names can collide with OS environment variables, local shell variables, or keys from another Vault path, causing Spring placeholder resolution to pick an unintended value.

## Correct Pattern

Import contextual Vault locations with explicit prefixes and the full Vault KV mount path, for example `vault://secret/database/path?prefix=db.` and `vault://secret/kafka/path?prefix=kafka.`. Reference the prefixed keys from application configuration, such as `${db.USER}`, `${db.PASSWORD}`, `${kafka.TRAFFIC_USERNAME}`, and `${kafka.TRAFFIC_PASSWORD}`.

## Reusable Insight

When Vault key names are controlled by another team or already deployed, avoid forcing secret renames as the first move. Use ConfigData import prefixes to create application-local namespaces around each imported path, and keep YAML placeholders pointed at the prefixed property names.

For explicit `spring.config.import` locations, do not rely on `spring.cloud.vault.kv.backend` to prefix the contextual path. Spring Cloud Vault reads the location path itself, including the KV mount. If the mount is `secret` and the secret path is `svc-dev/nfv-dev/postgresql`, import it as `vault://secret/svc-dev/nfv-dev/postgresql?prefix=db.`, not as `vault://nfv-dev/postgresql?prefix=db.` plus `spring.cloud.vault.kv.backend=secret`.

## Detection

Look for placeholders using generic names, especially `${USER}`, `${PASSWORD}`, `${TOKEN}`, `${SECRET}`, or duplicated keys imported from multiple Vault contexts. Also check whether local startup could inherit a same-named environment variable. If a runtime log shows a literal placeholder such as `${db.USER}` reaching a datasource or client library, compare each `spring.config.import` location against the full Vault path including the mount name.

## Verification

Add configuration tests that load application YAML with representative Vault-prefixed property sources and assert the final bound configuration uses the expected values. For Kafka dotted client properties, bind `spring.kafka` through Spring Boot's `Binder` and verify the resulting `KafkaProperties` maps contain the expected JAAS strings.
