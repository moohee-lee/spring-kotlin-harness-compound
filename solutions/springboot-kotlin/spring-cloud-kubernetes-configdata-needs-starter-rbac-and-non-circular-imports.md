---
title: Spring Cloud Kubernetes ConfigData에는 스타터, RBAC, 비순환 import가 필요하다
tags: [springboot-kotlin, kubernetes, configdata, helm]
scope: cross-project
status: active
principles: [configuration-ownership]
---

# Spring Cloud Kubernetes ConfigData에는 스타터, RBAC, 비순환 import가 필요하다

> 관련 공통 원칙: [설정 소유권과 로딩 시점](../../principles/ko/configuration-ownership.md)

## 적용 시점

Spring Boot service가 Kubernetes ConfigMap에서 non-secret configuration을, Spring Cloud Vault에서 secret을 읽고, Helm chart가 RBAC, service account, ConfigMap, volume wiring을 함께 제공할 때 적용한다.

## 피해야 할 방향

발견하려는 ConfigMap 안에만 `spring.config.import: kubernetes:`를 넣으면 안 된다. application은 ConfigMap을 읽기 전에 import가 필요하므로 bootstrap loop가 생긴다. RoleBinding만 만들고 Deployment가 bound service account를 쓰지 않는 것도 충분하지 않다. Vault CSI로 mount한 `application.yaml`과 direct `vault://` ConfigData import를 같은 setting에 섞는 것도 import order가 의도된 경우가 아니면 피한다.

operations가 ConfigMap을 standalone Kubernetes manifest로 관리하려는 경우에는 full `application.yml` body를 Helm values 안에 묻지 않는다. 그렇게 하면 application config lifecycle이 chart rendering에 결합되고 `kubectl apply -f configmap.yaml` workflow가 불편해진다.

## 권장 패턴

Kubernetes ConfigData import는 packaged application config, profile-specific config, 또는 explicit `SPRING_CONFIG_IMPORT` env var에서 제공한다. Kubernetes Java Client stack을 쓴다면 보통 `org.springframework.cloud:spring-cloud-starter-kubernetes-client-config` 하나만 추가한다. Pod는 읽으려는 namespace의 `configmaps`에 대해 `get`, `list`, `watch` 권한이 있는 service account로 실행한다.

non-secret은 ConfigMap, secret은 Vault에 둔다. direct Vault ConfigData import를 사용한다면 같은 setting에 대해 CSI secret-file mounting을 끈다.

ConfigMap을 Helm 밖에서 관리한다면 chart는 ConfigMap을 render하지 않는다. packaged Kubernetes profile에 service ConfigMap name을 hard-code하거나 values에는 ConfigMap name/reference만 두고 `SPRING_CLOUD_KUBERNETES_CONFIG_NAME`으로 전달한다. raw ConfigMap manifest는 Pod와 같은 namespace에 둔다. Kubernetes namespace와 Vault path 또는 Vault Kubernetes role은 서로 다른 설정 축으로 취급한다.

`spring.config.import: kubernetes:`와 `spring.cloud.kubernetes.config.sources`는 같은 early profile document에 둔다. import는 environment profile에 있고 source list는 별도 Kubernetes profile에 있으면 import가 source list binding보다 먼저 resolve되어 의도한 ConfigMap을 못 읽을 수 있다.

## 공통 원칙

Spring Cloud Kubernetes ConfigData에는 세 가지 독립 요구사항이 있다.

- classpath support
- early import activation
- Kubernetes API authorization

application build는 첫 번째를, Helm/deployment는 뒤 두 가지를 명시적으로 만족해야 한다.

## 점검 방법

다음을 확인한다.

- `spring.config.import: kubernetes:`가 ConfigMap 안에만 있는지
- `spring-cloud-starter-kubernetes-*-config` dependency가 빠졌는지
- `RoleBinding` subject와 `spec.serviceAccountName`이 다른지
- 같은 configuration에 Secrets Store CSI `application.yaml`과 `vault://` import가 동시에 켜졌는지
- standalone ConfigMap workflow가 목표인데 Helm values 아래 큰 `applicationYaml` block을 두는지
- rendered Helm namespace, external ConfigMap `metadata.namespace`, packaged `spring.cloud.kubernetes.config.namespace`가 Pod namespace와 일치하는지

list property의 indexed environment override도 확인한다. `SPRING_CLOUD_KUBERNETES_CONFIG_SOURCES_0_NAMESPACE` 하나만 있으면 profile-owned source list 전체를 대체해 추가 ConfigMap을 조용히 drop할 수 있다. indexed override를 피하거나 모든 source의 `_NAME`, `_NAMESPACE`를 완전한 list로 제공한다.

## 검증 방법

실제 chart와 values로 `helm lint`, `helm template`을 실행한다. rendered Deployment에서 `serviceAccountName`, `SPRING_CONFIG_IMPORT` 또는 packaged import strategy, RBAC가 맞는지 확인한다. Spring configuration test나 in-cluster smoke test로 Kubernetes ConfigMap property source와 Vault import가 local live value fallback 없이 resolve되는지 확인한다.

externally managed ConfigMap은 별도로 dry-run하고, Helm render에 ConfigMap object가 없으며 packaged profile이나 rendered Deployment가 expected ConfigMap name을 가리키는지 확인한다. `helm template ... -n <namespace>`로 namespace를 명시해 namespaced object가 같은 namespace로 render되는지도 assert한다. Deployment에 indexed `SPRING_CLOUD_KUBERNETES_CONFIG_SOURCES_*`가 있으면 expected source마다 `_NAME`, `_NAMESPACE`가 모두 있는지 확인한다.
