---
title: Helm value file은 단일 ConfigMap 템플릿을 여러 리소스로 만들지 못한다
tags: [springboot-kotlin, kubernetes, helm, argocd, configmap]
scope: cross-project
status: active
principles: [configuration-ownership]
---

# Helm value file은 단일 ConfigMap 템플릿을 여러 리소스로 만들지 못한다

> 관련 공통 원칙: [설정 소유권과 로딩 시점](../../principles/ko/configuration-ownership.md)

## 적용 시점

Spring Boot service가 Spring Cloud Kubernetes ConfigData로 여러 named ConfigMap을 읽고, ArgoCD Application이 별도 values repository의 여러 Helm value file을 공급할 때 적용한다. 많은 service chart는 top-level `configMap` value object 하나와 `templates/configmap.yaml` 하나만 제공한다.

## 피해야 할 방향

서로 다른 value file이 각각 `.Values.configMap.name`과 `.Values.configMap.files.application.yml`을 설정한다고 해서 Kubernetes ConfigMap 여러 개가 render된다고 기대하면 안 된다. Helm은 value file을 순서대로 merge하며, 나중 scalar와 같은 key의 map value가 앞 값을 덮어쓴다. ArgoCD Helm source에서 참조한 value file도 standalone Kubernetes manifest로 적용되지 않는다.

## 권장 패턴

chart가 ConfigMap 하나만 지원한다면 non-secret property를 하나의 ConfigMap에 모으고 Spring도 그 이름 하나만 읽게 한다. 여러 ConfigMap이 필요하면 chart에 `extraConfigMaps`나 `configMaps` 같은 명시적 list surface를 추가하고 각 item을 render한다. ConfigMap을 chart 밖에서 관리해야 한다면 Helm value file로만 두지 말고 ArgoCD의 별도 manifest source/path로 포함한다. values repository 안의 raw manifest를 쓰는 경우 nested directory를 포함해야 하면 별도 ArgoCD source가 manifest directory를 가리키고 `directory.recurse: true`를 설정해야 한다.

ConfigMap 변경이 Helm checksum rollout을 유발해야 한다면 ConfigMap과 Deployment가 같은 Helm render 안에 있어야 한다. values repository에는 파일을 분리해 두되 `.Values.configMaps.<name>.files` 같은 multi-ConfigMap surface로 넣고, ArgoCD `helm.valueFiles`에 포함한다. 같은 ConfigMap의 raw manifest source는 제거한다. Deployment pod template annotation에서 ConfigMap template output을 hash해 data 변경이 `spec.template.metadata.annotations` 변경으로 이어지게 한다.

## 공통 원칙

Helm value file은 chart template을 설정할 뿐, chart가 loop나 별도 value surface를 갖지 않으면 resource instance를 추가로 만들지 않는다. checksum annotation은 Deployment와 같은 Helm render에 포함된 template만 hash할 수 있다. ArgoCD가 별도 raw source로 적용한 ConfigMap은 그 hash에서 보이지 않는다.

## 점검 방법

`spring.cloud.kubernetes.config.sources[*].name`과 rendered Helm output을 비교한다. application은 여러 named ConfigMap을 기대하는데 `helm template` 결과가 `kind: ConfigMap` 하나뿐이라면 value file order와 `.Values.configMap` 아래 same-key override를 확인한다. ConfigMap이 external raw manifest라면 ArgoCD Application source에 manifest directory를 가리키는 non-Helm source와 필요한 recursion이 있는지 확인한다.

## 검증 방법

ArgoCD와 같은 value file order와 namespace로 `helm template`을 실행한다. rendered `kind: ConfigMap` 개수, expected `metadata.name`, `metadata.namespace`를 확인한 뒤 Spring placeholder나 RBAC를 디버깅한다. external raw manifest는 manifest directory를 별도로 parse/dry-run하고, ArgoCD Application이 그 directory를 standalone source로 포함하는지 확인한다.

checksum rollout은 rendered Deployment에 `checksum/configmap` annotation이 있는지 assert하고, harmless ConfigMap data 변경 후 checksum이 바뀌는지 확인한다.
