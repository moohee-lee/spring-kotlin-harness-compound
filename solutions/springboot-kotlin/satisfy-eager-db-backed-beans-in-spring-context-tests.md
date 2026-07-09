---
title: Spring context test는 eager DB-backed bean의 의존성을 명시적으로 만족시킨다
tags: [springboot-kotlin, testing, bean-lifecycle]
scope: cross-project
status: active
principles: [testing-contracts]
---

# Spring context test는 eager DB-backed bean의 의존성을 명시적으로 만족시킨다

> 관련 공통 원칙: [테스트는 계약을 검증해야 한다](../../principles/ko/testing-contracts.md)

## 적용 시점

Spring Boot bean이 lazy/runtime loading에서 database-backed port를 통한 eager constructor-time loading으로 의도적으로 바뀔 때 적용한다. required system property, tenant metadata, region identifier를 application-scoped holder에 로딩하는 경우가 대표적이다.

## 피해야 할 방향

holder나 service unit test만 수정하고 disabled runner 때문에 full Spring context test는 관련 없다고 가정하면 안 된다. disabled `ApplicationRunner`는 ordinary singleton service와 그 constructor dependency가 context startup 중 생성되는 것을 막지 않는다.

## 권장 패턴

eager-loading production contract를 명시적으로 둔다. holder는 output port를 주입받고 required value가 없으면 bean creation을 실패시킨다. Spring context test에서는 test 목적에 맞는 가장 작은 input으로 해당 dependency를 만족시킨다.

- persistence를 테스트하지 않는다면 `@TestConfiguration`의 primary port fake를 사용한다.
- real persistence adapter를 테스트해야 한다면 test database seed data를 넣는다.

## 공통 원칙

DB read를 bean construction으로 옮기면 blast radius가 runtime use case에서 bean을 생성하는 모든 Spring context로 커진다. test profile, disabled startup runner, mocked external client는 constructor-time requirement를 제거하지 않는다.

## 점검 방법

eager DB-backed bean 변경 후 최소 하나의 full `@SpringBootTest` context test를 실행한다. context startup 중 `BeanCreationException`, `UnsatisfiedDependencyException`, holder의 missing-configuration exception이 발생하는지 확인한다.

## 검증 방법

successful eager loading과 missing-value failure를 검증하는 focused holder test를 추가하거나 수정한다. 그다음 해당 bean을 생성하는 Spring context test와 영향을 받는 service test를 실행한다. full test suite에는 eager bean의 context startup failure가 남아 있으면 안 된다.
