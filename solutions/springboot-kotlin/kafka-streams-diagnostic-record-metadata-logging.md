---
title: Kafka Streams 진단 로그에는 record metadata를 포함한다
tags: [springboot-kotlin, kafka-streams, logging, observability]
scope: cross-project
status: active
principles: [kafka-streams-operations]
---

# Kafka Streams 진단 로그에는 record metadata를 포함한다

> 관련 공통 원칙: [Kafka Streams 운영 계약](../../principles/ko/kafka-streams-operations.md)

## 적용 시점

Kafka Streams를 사용하는 Spring Boot Kotlin service에서 mapping, join, publish decision 실패 뒤에 어떤 Kafka record가 원인인지 incident log로 추적해야 할 때 적용한다. `stream()` 이후의 DSL operator는 보통 topic, partition, offset을 직접 노출하지 않는다.

## 피해야 할 방향

변환된 DTO나 exception만 logging하면 문제 record를 replay하거나 조사하기 어렵다. business DTO에 offset field를 임의로 추가하는 것도 transport metadata를 domain-shaped message model로 새게 만든다.

## 권장 패턴

input topic을 읽은 직후 `FixedKeyProcessor`와 `FixedKeyProcessorContext.recordMetadata()` 기반의 작은 processor로 Kafka metadata를 capture한다. value는 nullable metadata와 원본 message summary를 담는 internal processing record로 감싼다. 이후 DSL step은 external message contract를 바꾸지 않고도 WARN/ERROR log에 `topic`, `partition`, `offset`을 포함할 수 있다.

high-volume per-record payload/flow log는 TRACE에 둔다. raw record receipt, candidate extraction, join observation, produced-record trace가 여기에 해당한다. 일반 개발 중 켜둘 수 있는 낮은 volume의 diagnostic state만 DEBUG에 둔다. expected unmatched reference data는 WARN이 아니라 낮은 level로 둔다. record를 변환할 수 없고 exception을 rethrow할 때 ERROR를 사용한다.

## 공통 원칙

Kafka 위치 정보는 business data가 아니라 diagnostic context다. boundary에서 붙이고 internal processing 동안 함께 운반하되, source payload와 joined reference data는 record volume에 맞는 log level과 compact summary로 남긴다.

## 점검 방법

Kafka Streams topology에서 filter/map/join 실패 log가 "skipped"나 "failed"만 말하고 source topic/partition/offset 또는 bad input을 식별할 source field를 남기지 않는지 찾는다.

## 검증 방법

`TopologyTestDriver`와 Logback `ListAppender`를 사용해 skipped/mismatched record가 expected Kafka metadata와 compact source data를 logging하는지 검증한다. production logging은 INFO를 유지하고 development profile에서만 DEBUG를 켜는 profile configuration test도 추가한다.
