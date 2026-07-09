---
title: Kafka Streams GlobalKTable state.dir를 영속화한다
tags: [springboot-kotlin, kafka-streams, globalktable, state-dir]
scope: cross-project
status: active
principles: [kafka-streams-operations]
---

# Kafka Streams GlobalKTable state.dir를 영속화한다

> 관련 공통 원칙: [Kafka Streams 운영 계약](../../principles/ko/kafka-streams-operations.md)

## 적용 시점

Kafka Streams `GlobalKTable` join을 사용하는 Spring Boot service가 재시작 때마다 table topic 전체를 replay하는 것처럼 보일 때 적용한다. 원인은 Streams local state directory가 temporary container path에 있거나, startup에 cleanup되거나, persistent storage로 mount되지 않는 경우가 많다. Kafka Streams는 `state.dir`가 재시작 사이에 살아 있으면 `GlobalKTable` state와 checkpoint offset을 local에 유지한다.

## 피해야 할 방향

Kafka Streams state directory를 확인하기 전에 별도의 application-managed disk cache나 offset table부터 만들면 안 된다. Kafka Streams 밖에 state를 중복 저장하면 offset/store 불일치 위험과 복구 code가 추가된다.

## 권장 패턴

Spring Boot 설정으로 안정적인 Kafka Streams state directory를 지정한다.

```yaml
spring:
  kafka:
    streams:
      state-dir: /var/lib/kcp-network-consumer/kafka-streams
      cleanup:
        on-startup: false
        on-shutdown: false
```

container 환경에서는 해당 path를 persistent volume에 mount한다. `spring.kafka.streams.application-id`는 안정적으로 유지한다. application id는 local state path의 일부이므로 바뀌면 새 Streams application으로 취급된다. `spring.kafka.streams.properties.auto.offset.reset: earliest`는 local checkpoint가 없는 cold start fallback으로만 사용한다.

application id를 바꿔야 한다면 rename이 아니라 새 Kafka Streams application으로 다룬다. 새 group ACL을 준비하고, `state.dir/<new-application-id>` 아래 새 state directory가 생긴다고 예상하며, source topic별 시작 offset을 startup 전에 결정한다. replay 가능한 `GlobalKTable` reference topic과 고용량 fact stream을 함께 쓰는 topology라면 global consumer는 처음부터 rebuild할 수 있게 하되, fact stream replay는 명시적 backfill이 아닌 한 막아야 한다.

container가 non-root UID로 실행된다면 persistent volume을 정확히 `state-dir` path에 직접 mount하지 않는 편이 안전하다. Kafka Streams는 startup 중 `state.dir` permission을 설정하고, PVC mount root는 `fsGroup`이 group write를 주더라도 root 소유인 경우가 많다. volume은 parent path에 mount하고 application UID가 child directory를 만들게 하거나, init container로 ownership/chmod를 준비한다.

## 공통 원칙

`GlobalKTable`은 `state.dir/<application-id>/global` 아래 replicated table state를 저장하고 recovery용 checkpoint offset을 쓴다. 재시작 때 checkpoint가 살아 있으면 global consumer는 저장된 offset부터 seek한다. directory나 checkpoint가 없으면 source topic 처음부터 bootstrap해야 한다.

## 점검 방법

profile 또는 base YAML에서 `spring.kafka.streams.state-dir`가 없거나 default `/tmp/kafka-streams`를 쓰거나 cleanup on startup/shutdown이 켜져 있는지 확인한다. Kubernetes에서는 configured path가 `emptyDir`나 image filesystem이 아니라 persistent volume인지 확인한다. Deployment `volumeMounts[].mountPath`와 `spring.kafka.streams.state-dir`가 같고 pod가 non-root로 실행되면 `StateDirectory` permission error가 날 수 있다.

migration 작업에서는 `application-id` 변경과 `auto.offset.reset=earliest`가 함께 있는지 특별히 확인한다. 새 application id는 committed offset이 없으므로 retained source record를 output topic으로 replay할 수 있다.

## 검증 방법

configuration test로 `spring.kafka`를 bind하고 다음을 assert한다.

- `KafkaProperties.streams.stateDir`가 configured path와 같다.
- `KafkaProperties.streams.cleanup.onStartup`이 `false`다.
- `KafkaProperties.streams.cleanup.onShutdown`이 `false`다.
- cold start가 필요하다면 `auto.offset.reset` 설정이 의도대로 남아 있다.

runtime 검증에서는 app을 시작해 `GlobalKTable` bootstrap이 끝나게 하고, graceful shutdown 후 state directory에 application id와 global store checkpoint file이 있는지 확인한 다음 재시작한다.
