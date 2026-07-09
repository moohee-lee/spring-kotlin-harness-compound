---
title: Kubernetes 종료 grace와 Kafka Streams close timeout을 맞춘다
tags: [springboot-kotlin, kubernetes, kafka-streams, graceful-shutdown]
scope: cross-project
status: active
principles: [kafka-streams-operations, runtime-boundaries]
---

# Kubernetes 종료 grace와 Kafka Streams close timeout을 맞춘다

> 관련 공통 원칙: [Kafka Streams 운영 계약](../../principles/ko/kafka-streams-operations.md), [런타임 경계와 컨테이너 계약](../../principles/ko/runtime-boundaries.md)

## 적용 시점

Kubernetes에서 실행되는 Spring Boot Kotlin service가 Kafka Streams를 사용하고, Deployment `strategy.type: Recreate` 또는 persistent Kafka Streams state directory를 함께 사용할 때 적용한다. Kafka Streams 종료는 Spring lifecycle과 `StreamsBuilderFactoryBean.closeTimeout`의 영향을 받고, Kubernetes는 termination 시작 후 container가 살아 있을 수 있는 시간을 제한한다.

## 피해야 할 방향

application code에서 Kafka Streams close timeout을 60초 이상으로 설정했는데 Kubernetes `terminationGracePeriodSeconds`를 기본값 30초로 두면 안 된다. `preStop` sleep만 추가하고 termination grace period를 늘리지 않는 것도 잘못이다. Kubernetes는 `preStop` hook이 실행되기 전에 grace countdown을 시작하므로 hook 시간도 같은 종료 budget을 소비한다.

## 권장 패턴

Kubernetes grace period를 Spring shutdown phase timeout, Kafka Streams close timeout, `preStop` drain delay의 합보다 크게 설정한다. inbound HTTP에 대해 Spring Boot graceful shutdown을 활성화하고, `spring.lifecycle.timeout-per-shutdown-phase`를 Kafka Streams close timeout과 맞춘다. state를 영속화한다면 Kafka Streams `cleanup.on-shutdown=false`를 유지한다. `preStop`은 endpoint나 load balancer drain delay 용도로만 사용하고, Kafka Streams 종료의 주 메커니즘으로 삼지 않는다.

## 공통 원칙

Graceful shutdown은 Kubernetes, Spring Boot, Kafka Streams 사이의 budget alignment 문제다. 가장 작은 timeout이 전체 종료 품질을 결정한다. Kubernetes가 Spring/Kafka Streams close 완료 전에 container를 죽이면 offset과 local state가 깨끗하게 정리되지 않을 수 있다.

## 점검 방법

다음 설정을 함께 확인한다.

- `strategy.type: Recreate`
- `terminationGracePeriodSeconds`
- `server.shutdown`
- `spring.lifecycle.timeout-per-shutdown-phase`
- `StreamsBuilderFactoryBean.setCloseTimeout`
- Kafka Streams `state-dir`
- `cleanup.on-shutdown`

Kubernetes grace가 application close timeout보다 작거나 같으면 수정 대상이다.

## 검증 방법

Helm을 render해서 Pod spec에 의도한 `terminationGracePeriodSeconds`와 lifecycle hook이 있는지 확인한다. rollout 또는 manual pod deletion 중에는 process가 SIGTERM을 받고, readiness가 traffic을 중단하며, Kafka Streams가 정상 shutdown log를 남기고, SIGKILL/exit 137이 없어야 한다. 새 pod가 persisted state directory에서 full rebuild 없이 복구되는지도 확인한다.
