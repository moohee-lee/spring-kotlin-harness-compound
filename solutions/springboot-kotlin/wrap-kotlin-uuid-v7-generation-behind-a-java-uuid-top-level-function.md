---
title: UUID v7 생성은 java.util.UUID wrapper 함수 뒤에 숨긴다
tags: [springboot-kotlin, kotlin, uuid-v7]
scope: cross-project
status: active
principles: [adapter-boundaries]
---

# UUID v7 생성은 java.util.UUID wrapper 함수 뒤에 숨긴다

> 관련 공통 원칙: [어댑터 경계와 재사용 구조](../../principles/ko/adapter-boundaries.md)

## 적용 시점

Spring Boot Kotlin service가 저장하거나 노출하는 ID type은 `java.util.UUID`로 유지하면서 새로 생성하는 identifier는 시간 정렬성이 좋은 UUID v7을 쓰고 싶을 때 적용한다.

## 피해야 할 방향

service, worker, filter, adapter 곳곳에서 `UUID.randomUUID()`를 직접 호출하면 v4 생성이 project 전체에 흩어진다. 각 call site를 low-level bit manipulation이나 third-party UUID library로 바꾸는 것도 Kotlin stdlib가 UUID v7 생성을 제공하는 상황에서는 불필요한 code와 dependency surface를 만든다.

## 권장 패턴

common utility package에 `uuidV7(): java.util.UUID` 같은 project-local Kotlin top-level function을 하나 만든다. 구현은 `kotlin.uuid.Uuid.generateV7().toJavaUuid()`를 사용하고 `@OptIn(ExperimentalUuidApi::class)`는 wrapper 내부에만 둔다.

단순 ID 생성 지점에서는 이 함수를 직접 호출한다. worker lease/token flow처럼 deterministic concurrency나 retry test를 위해 실제 복잡도를 줄여주는 경우에만 injectable generator lambda를 유지한다.

## 공통 원칙

Kotlin UUID API는 UUID v7 생성을 제공하면서도 Spring/JPA/jOOQ/WebFlux codebase의 나머지 type은 `java.util.UUID`로 유지할 수 있게 한다. 작은 wrapper는 experimental API annotation과 conversion detail을 domain/application code 밖으로 숨긴다.

## 점검 방법

production code에서 `UUID.randomUUID`, `UUID::randomUUID`, direct UUID library call을 검색한다. generated ID, lock token, trace ID, outbox ID, callback idempotency ID는 의도적으로 다른 UUID version이 필요한 경우를 제외하고 shared generator를 거쳐야 한다. 단순 function call을 전달하기만 하는 default generator/clock constructor parameter도 실제 test/runtime configuration 필요가 없다면 제거한다.

## 검증 방법

생성된 UUID가 `version() == 7`이고 RFC 4122 variant `variant() == 2`인지 test한다. utility function만이 아니라 consuming default도 test해 future call site가 v4로 되돌아가지 않게 한다. 단순한 service에는 review feedback이 불필요한 injection을 지적한 경우에만 lightweight constructor-shape 또는 wiring test를 추가한다.
