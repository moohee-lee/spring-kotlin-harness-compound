---
title: Spring Cloud Kubernetes ConfigData needs starter RBAC and non circular imports
tags: [springboot-kotlin, kubernetes, configdata, helm]
scope: cross-project
status: draft
---

# Spring Cloud Kubernetes ConfigData needs starter RBAC and non circular imports

## Context
Spring Boot services may load non-secret configuration from Kubernetes
ConfigMaps with Spring Cloud Kubernetes ConfigData and secrets from Spring
Cloud Vault. Helm charts often provide RBAC, service account, ConfigMap, and
volume wiring at the same time, so it is easy to mix runtime mechanisms.

## Wrong Direction
Do not put `spring.config.import: kubernetes:` only inside the ConfigMap that
is supposed to be discovered through that same import. That creates a
bootstrap loop: the application needs the import before it can load the
ConfigMap. Also do not assume a chart RoleBinding is enough if the Deployment
does not use the bound service account, and do not mix Vault CSI mounted
`application.yaml` with direct `vault://` ConfigData imports unless the import
order is intentional. If operations want ConfigMaps managed as standalone
Kubernetes manifests, do not bury the full `application.yml` body inside Helm
values; that couples application config lifecycle to chart rendering and makes
manual `kubectl apply -f configmap.yaml` workflows awkward.

## Correct Pattern
Provide the Kubernetes ConfigData import from packaged application config,
profile-specific config, or an explicit `SPRING_CONFIG_IMPORT` env var. Add one
Spring Cloud Kubernetes config starter only, usually
`org.springframework.cloud:spring-cloud-starter-kubernetes-client-config` for
the Kubernetes Java Client stack. Bind the Pod to a service account that has
`get`, `list`, and `watch` on `configmaps` in the namespace being read. Keep
non-secrets in ConfigMaps and secrets in Vault; if using direct Vault
ConfigData imports, disable CSI secret-file mounting for the same settings.
When the ConfigMap is managed outside Helm, render no ConfigMap from the chart:
either hard-code the service ConfigMap name in the packaged Kubernetes profile
or put only a ConfigMap name/reference in values and pass it to
`SPRING_CLOUD_KUBERNETES_CONFIG_NAME`. Keep the raw ConfigMap manifest in the
same namespace as the Pod. Treat Kubernetes namespace and Vault path or Vault
Kubernetes role as separate configuration axes; do not infer one from the other
just because they contain environment-like names.

## Reusable Insight
Spring Cloud Kubernetes ConfigData has three independent requirements:
classpath support, early import activation, and Kubernetes API authorization.
Helm needs to satisfy the latter two explicitly; application code/build needs
to satisfy the first.

## Detection
Look for `spring.config.import: kubernetes:` only in a ConfigMap, missing
`spring-cloud-starter-kubernetes-*-config` dependency, `RoleBinding` subjects
that do not match `spec.serviceAccountName`, or values files that enable both
Secrets Store CSI `application.yaml` and Spring Cloud Vault `vault://` imports
for the same configuration. Also look for large `applicationYaml` blocks under
Helm values when the intended operational workflow is a standalone ConfigMap
manifest. Also compare the rendered Helm namespace, external ConfigMap
`metadata.namespace`, and packaged `spring.cloud.kubernetes.config.namespace`
default; they should agree with the Pod namespace, independently of Vault
secret paths or Vault role names.

## Verification
Run `helm lint` and `helm template` with the exact chart and values. In the
rendered Deployment, confirm `serviceAccountName`, `SPRING_CONFIG_IMPORT` or
packaged import strategy, and RBAC all line up. Run a Spring configuration test
or an in-cluster smoke test and verify logs show Kubernetes ConfigMap property
sources and Vault imports resolving without falling back to live local values.
For externally managed ConfigMaps, separately dry-run the ConfigMap manifest,
confirm the Helm render contains no ConfigMap object, and confirm the packaged
profile or rendered Deployment points at the expected ConfigMap name.
Render Helm with the actual namespace flag, such as `helm template ... -n
<namespace>`, and assert all namespaced Kubernetes objects render into that
namespace.
