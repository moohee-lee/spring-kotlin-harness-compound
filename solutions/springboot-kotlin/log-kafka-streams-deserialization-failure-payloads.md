---
title: Kafka Streams 역직렬화 실패 payload를 남긴다
tags: [springboot-kotlin, kafka-streams, logging, deserialization]
scope: cross-project
status: active
principles: [kafka-streams-operations]
---

# Kafka Streams 역직렬화 실패 payload를 남긴다

> 관련 공통 원칙: [Kafka Streams 운영 계약](../../principles/ko/kafka-streams-operations.md)

## 적용 시점

Kafka Streams를 사용하는 Spring Boot Kotlin service에서 JSON deserialization이 business logic 진입 전에 실패할 수 있을 때 적용한다. 기본 `LogAndFailExceptionHandler`는 topic, partition, offset은 남기지만 raw message payload는 남기지 않는다. `GlobalKTable` join에서는 materialized state store를 읽는 중 deserialization error가 드러날 수 있어 source-level handler만으로는 bad value를 확인하기 어렵다.

## 피해야 할 방향

incident에 실제 message body가 필요한데 default Kafka Streams deserialization handler만 믿으면 안 된다. DSL join 이후 business-field log를 추가해도 record가 그 operator에 도달하기 전에 실패할 수 있다.

이미 `GlobalKTable`로 등록된 topic을 source offset logging 목적으로 다시 일반 `KStream`으로 등록해서도 안 된다. Kafka Streams는 duplicate source-topic registration을 거부한다.

## 권장 패턴

parsing 위치에 맞는 diagnostic layer를 둔다.

1. custom `DeserializationExceptionHandler`를 구성해 topic/partition/offset과 raw key/value preview를 logging한다. fail-fast 동작을 유지해야 하면 `FAIL`을 반환한다.
2. state store나 reference table을 backing하는 JSON `Serde`는 deserializer decorator로 감싼다. `SerializationException`이 나면 configured source topic, deserializer topic/store name, raw UTF-8 payload를 남기고 다시 throw한다.
3. Kafka contract가 `String` key/value이고 JSON parse를 DSL code에서 직접 한다면 source와 materialized value를 `String`으로 유지하고 processing lambda 안에서 project JSON extension을 호출한다. `KStream` source는 parsing 전에 processor로 `recordMetadata()`를 capture한다. `GlobalKTable` source는 joiner가 materialized value만 보고 source offset은 보지 못한다는 점을 기억한다. malformed raw table update는 `TimestampExtractor` 같은 source hook에서 logging하거나 더 풍부한 metadata가 필요하면 custom global store processor로 옮긴다.

payload preview는 예를 들어 4 KB로 제한하고 newline을 escape해서 single-event diagnostic log로 유지한다.

## 공통 원칙

failure surface는 source-record deserialization과 state-store value deserialization으로 나뉜다. Kafka metadata는 source path에서 가장 강하고, serde boundary의 raw value logging은 state-store path에서도 동작한다.

String serde를 사용해 serde-time failure를 피하면 parser는 application code로 이동한다. 이 경우 skip path는 `mapValues`, `selectKey`, joiner에서 throw하지 말고 빈 결과를 반환해야 한다.

## 점검 방법

stack trace에 `JacksonJsonDeserializer`, `ValueAndTimestampDeserializer`, `StateSerdes.valueFrom`, `KTableSourceValueGetter`가 보이는데 log에는 byte array만 있고 JSON payload가 없다면 보강 대상이다. `LogAndFailExceptionHandler`만 구성되어 있으면 bad message body는 application log에 없을 가능성이 높다.

String serde topology에서는 DSL lambda가 JSON parse 후 null parse result에서 throw하는지 확인한다. 이 exception은 processing error로 취급되어 Kafka Streams를 멈출 수 있다.

## 검증 방법

custom handler에 `ConsumerRecord<byte[], byte[]>`를 넘기는 unit test를 만들고 key/value preview log를 assert한다. wrapped serde를 malformed JSON으로 직접 호출해 topic argument가 materialized store name이어도 payload가 logging되는지 확인한다. Kubernetes/runtime profile이 custom handler를 사용하는지도 property test로 확인한다.

String serde topology는 `TopologyTestDriver`와 `StringSerializer` input으로 malformed source/reference value를 넣고 topic, partition, offset, raw value가 logging되며 이후 valid record는 계속 output을 만드는지 확인한다.
