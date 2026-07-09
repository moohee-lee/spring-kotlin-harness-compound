---
title: Spring Boot Kafka dotted client property는 Binder로 검증한다
tags: [springboot-kotlin, kafka, configuration, binder]
scope: cross-project
status: active
principles: [configuration-ownership, kafka-streams-operations]
---

# Spring Boot Kafka dotted client property는 Binder로 검증한다

> 관련 공통 원칙: [설정 소유권과 로딩 시점](../../principles/ko/configuration-ownership.md), [Kafka Streams 운영 계약](../../principles/ko/kafka-streams-operations.md)

## 적용 시점

Spring Boot Kafka configuration이 raw Kafka client property를 `spring.kafka.properties`, `spring.kafka.producer.properties`, `spring.kafka.streams.properties` 아래에 저장할 때 적용한다. Kafka Streams는 `main.consumer.sasl.jaas.config`, `global.consumer.sasl.jaas.config`, `producer.sasl.jaas.config`, `admin.sasl.jaas.config` 같은 dotted client prefix도 지원한다.

## 피해야 할 방향

YAML text나 flattened `YamlPropertySourceLoader` key만 확인하면 binder-level issue를 놓칠 수 있다. dotted key가 file에 보여도 runtime에서 Spring Boot가 사용하는 `KafkaProperties` map shape로 bind되지 않을 수 있다.

## 권장 패턴

Spring Boot 4에서는 loaded environment를 `org.springframework.boot.kafka.autoconfigure.KafkaProperties`로 bind하고 resulting map에 정확한 Kafka client key가 들어 있는지 assert한다. Spring Boot 4 package는 예전 `org.springframework.boot.autoconfigure.kafka`가 아니라 `org.springframework.boot.kafka.autoconfigure`다.

## 공통 원칙

Dotted Kafka client property key는 두 층을 모두 검증할 때 안전하다.

- config file에 intended key와 default가 있다.
- `Binder.get(environment).bind("spring.kafka", KafkaProperties::class.java)` 결과 map에 `security.protocol`, `main.consumer.sasl.jaas.config`, `producer.sasl.jaas.config` 같은 key가 있다.

## 점검 방법

SCRAM/SASL, per-client Kafka Streams credential, key 자체에 dot이 들어가는 Kafka property를 추가할 때 이 pattern을 적용한다.

## 검증 방법

focused configuration test에서 `application.yaml`을 load하고 `StandardEnvironment`에 property source를 추가한다. `spring.kafka`를 `KafkaProperties`로 bind한 뒤 common, producer, streams property map이 expected key를 포함하는지 assert한다.
