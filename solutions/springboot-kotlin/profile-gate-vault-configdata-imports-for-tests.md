---
title: Profile gate Vault ConfigData imports for tests
tags: [springboot-kotlin, vault, configdata, testing]
scope: cross-project
status: active
---

# Profile gate Vault ConfigData imports for tests

## Context

Spring Boot services that add Spring Cloud Vault through `spring.config.import` often also have `@SpringBootTest` or integration tests with a `test` profile and testcontainers-backed datasource configuration. Those tests should not depend on live Vault access.

## Wrong Direction

Putting `spring.config.import: vault://...` in the base application document and only adding `spring.cloud.vault.enabled=false` in `application-test.yaml` can still allow Vault ConfigData processing to happen before the test profile disables Vault. Tests may pass on a developer machine with valid credentials but perform unintended network calls.

## Correct Pattern

Prefer placing Vault ConfigData imports and Vault authentication settings in a profile-specific file such as `application-dev.yaml` or `application-local.yaml` when the project uses profile files:

```yaml
spring:
  config:
    import:
      - vault:///service/database?prefix=db.
```

If a single `application.yaml` must be used, place the imports in a profile-gated YAML document, for example:

```yaml
---
spring:
  config:
    activate:
      on-profile: "!test"
    import:
      - vault:///service/database?prefix=db.
```

Keep `spring.cloud.vault.enabled: false` in `application-test.yaml` as an explicit guard, but do not rely on it as the only isolation mechanism.

## Reusable Insight

ConfigData imports are part of early environment construction. Test isolation is more reliable when the import location itself is inactive for the test profile, not merely when a later Vault property says Vault is disabled.

## Detection

After adding Vault imports, run a full Spring context test and scan logs for Vault HTTP calls such as login, secret reads, or `auth/token/revoke-self`. A passing test suite can still be wrong if it contacted live Vault.

## Verification

Run the full test suite with the `test` profile active and confirm the output contains no Vault HTTP calls. Add configuration tests that assert base `application.yaml` has no Vault import, the profile-specific application file contains the Vault imports, and `application-test.yaml` contains `spring.cloud.vault.enabled: false`.
