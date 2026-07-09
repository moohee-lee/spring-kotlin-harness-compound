---
title: 프로필 소유 인프라 설정은 base YAML에서 분리한다
tags: [springboot-kotlin, configuration, profiles, secrets]
scope: cross-project
status: active
principles: [configuration-ownership]
---

# 프로필 소유 인프라 설정은 base YAML에서 분리한다

> 관련 공통 원칙: [설정 소유권과 로딩 시점](../../principles/ko/configuration-ownership.md)

## 적용 시점

Spring Boot service가 base `application.yaml`과 `application-dev.yaml`, `application-local.yaml`, `application-test.yaml` 같은 profile-specific file을 함께 사용할 때 적용한다. datasource, Kafka, Vault 같은 infrastructure endpoint와 credential은 대부분 profile마다 다르다.

## 피해야 할 방향

base `application.yaml`에 `${KAFKA_BOOTSTRAP_SERVERS:localhost:19092}` 같은 placeholder로 datasource/Kafka endpoint를 두면 안 된다. base 설정이 environment-specific이 되고, shell environment variable이 profile 의도를 조용히 override할 수 있다. 어느 profile이 어떤 infrastructure contract를 소유하는지도 흐려진다.

## 권장 패턴

base `application.yaml`에는 모든 profile에서 참인 application setting만 둔다. datasource, Kafka client property, Kafka topic, Vault import, credential placeholder는 `application-{profile}.yaml`로 옮긴다. non-secret endpoint와 mode는 profile file에 literal value로 두고, secret에만 Vault-prefixed placeholder를 사용한다.

Kubernetes ConfigMap이 profile-specific runtime configuration source라면 해당 ConfigMap은 owning profile file에서 생성하거나 curate한다. datasource JDBC URL, Kafka bootstrap server 같은 non-secret runtime endpoint는 ConfigMap의 embedded `application.yml`에 직접 두고, Deployment environment variable은 pod metadata, bootstrap toggle, application profile contract 밖의 값으로 제한한다.

## 공통 원칙

Profile file이 infrastructure wiring의 source of truth여야 한다. Environment variable은 secret이나 명시적 deployment parameter에는 유용하지만, shared YAML 안의 `${ENV:default}` fallback은 review와 runtime behavior를 어렵게 만든다.

## 점검 방법

base `application.yaml`에서 `spring.datasource`, `spring.kafka`, `app.kafka`, `${...:...}` fallback placeholder를 검색한다. 발견하면 정말 profile-independent인지 확인한다. datasource와 Kafka 값은 대부분 profile 소유다.

Kubernetes deployment에서는 Helm values와 rendered Deployment에서 `DB_JDBC_URL`, `KAFKA_BOOTSTRAP_SERVERS` 같은 app-owned endpoint env를 검색한다. 같은 data가 ConfigMap-backed profile에 속한다면 container env가 아니라 ConfigMap으로 옮긴다.

## 검증 방법

base와 profile YAML을 분리해서 load하는 configuration test를 둔다. base YAML에는 profile-owned infrastructure key가 없고, 각 profile file에는 expected literal datasource/Kafka value와 Vault-prefixed secret placeholder가 있는지 assert한다. test profile별 Spring context test도 실행한다.

ConfigMap-backed profile은 raw ConfigMap manifest와 embedded `application.yml`을 parse해 non-secret endpoint가 ConfigMap에 있고 environment-specific Helm values에는 없는지 확인한다.
