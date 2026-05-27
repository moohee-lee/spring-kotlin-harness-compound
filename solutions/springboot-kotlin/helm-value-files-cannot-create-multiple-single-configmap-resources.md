---
title: Helm value files cannot create multiple single ConfigMap resources
tags: [springboot-kotlin, kubernetes, helm, argocd, configmap]
scope: cross-project
status: draft
---

# Helm value files cannot create multiple single ConfigMap resources

## Context
Spring Boot services may use Spring Cloud Kubernetes ConfigData to load more
than one named ConfigMap, while an ArgoCD Application supplies multiple Helm
value files from a separate values repository. Many service charts expose one
top-level `configMap` value object and one `templates/configmap.yaml`.

## Wrong Direction
Do not expect separate value files that each set `.Values.configMap.name` and
`.Values.configMap.files.application.yml` to render separate Kubernetes
ConfigMaps. Helm merges value files in order; later scalar and same-key map
values override earlier values. Also do not assume a value file referenced from
an ArgoCD Helm source is applied as a standalone Kubernetes manifest.

## Correct Pattern
If the chart supports only one ConfigMap, either put all non-secret properties
into that one ConfigMap and configure Spring to read only that name, or extend
the chart with an explicit list such as `extraConfigMaps`/`configMaps` and
render every item. If a ConfigMap must be managed outside the chart, include it
as an actual ArgoCD manifest source/path rather than only as a Helm value file.
For value-repository raw manifests, point a separate ArgoCD source at the
manifest directory and set `directory.recurse: true` when nested directories
such as `configmaps/order/*.yaml` should be included.

## Reusable Insight
Helm value files configure chart templates; they do not create additional
resource instances unless the chart has a template loop or separate value
surface for those instances.

## Detection
Compare `spring.cloud.kubernetes.config.sources[*].name` with the rendered
Helm output. If the application expects multiple named ConfigMaps but
`helm template` shows only one `kind: ConfigMap`, inspect value file order and
same-key overrides under `.Values.configMap`. If the ConfigMaps are external
raw manifests, inspect the ArgoCD Application sources for a non-Helm source
with `path` set to the manifest directory and recursion enabled when needed.

## Verification
Run `helm template` with the exact ArgoCD value file order and namespace. Count
the rendered `kind: ConfigMap` objects and confirm each expected
`metadata.name` and `metadata.namespace` exists before debugging Spring
placeholder resolution or RBAC. For external raw manifests, parse or dry-run
the manifest directory separately and verify the ArgoCD Application includes
that directory as a standalone source.
