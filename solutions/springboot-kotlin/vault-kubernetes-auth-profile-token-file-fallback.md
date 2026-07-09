---
title: vault-kubernetes-auth-profile-token-file-fallback
tags: [springboot-kotlin, vault, kubernetes, configdata, profiles]
scope: cross-project
status: active
---

# Vault Kubernetes auth profile token file fallback

## Context
Spring Boot services can import Vault secrets with Spring Cloud Vault ConfigData and Kubernetes authentication. A service may have multiple runtime profiles: one for local execution with an exported service account token file, and another for Kubernetes execution where the pod-mounted default token path exists.

## Wrong Direction
Verifying Vault with `curl` or the Vault CLI and then assuming Spring will use the same token file for every active profile. If the active profile file does not set `spring.cloud.vault.kubernetes.service-account-token-file`, Spring Cloud Vault falls back to `/var/run/secrets/kubernetes.io/serviceaccount/token`, which only exists in a Kubernetes pod.

## Correct Pattern
Keep Vault import paths and Kubernetes authentication settings in each profile that can activate Vault. For profiles used both locally and in Kubernetes, use an explicit token file fallback:

```yaml
spring:
  cloud:
    vault:
      authentication: kubernetes
      kubernetes:
        role: <vault-role>
        kubernetes-path: <auth-mount>
        service-account-token-file: ${VAULT_K8S_TOKEN_FILE:/var/run/secrets/kubernetes.io/serviceaccount/token}
```

If a profile must be Kubernetes-only, document that local runs should activate the local profile instead.

## Reusable Insight
Direct Vault login success proves the role, auth mount, policy, and secret path can work. It does not prove the Spring application is using the same active profile or token file path.

## Detection
When Vault values are missing or startup fails during ConfigData processing, compare:

- `spring.profiles.active` and the loaded `application-{profile}.yaml`.
- `spring.config.import` entries for the active profile.
- `spring.cloud.vault.kubernetes.service-account-token-file` after placeholder resolution.
- The stack trace for fallback to `/var/run/secrets/kubernetes.io/serviceaccount/token` on a non-Kubernetes machine.

## Verification
Run a minimal Spring ConfigData probe or context startup with the same active profile and print only presence booleans for expected Vault-derived properties. Separately, call Vault login and secret-read endpoints without printing token or secret values to isolate external Vault access from Spring profile binding.
