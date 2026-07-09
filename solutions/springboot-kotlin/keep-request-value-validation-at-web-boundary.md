---
title: Request 값 검증은 web boundary에 둔다
tags: [springboot-kotlin, validation, hexagonal-architecture]
scope: cross-project
status: active
principles: [adapter-boundaries]
---

# Request 값 검증은 web boundary에 둔다

> 관련 공통 원칙: [어댑터 경계와 재사용 구조](../../principles/ko/adapter-boundaries.md)

## 적용 시점

Spring Boot Kotlin service가 hexagonal architecture를 따르고, inbound WebFlux handler가 request body, path variable, query parameter를 request DTO로 bind한 뒤 application service를 호출할 때 적용한다.

## 피해야 할 방향

handler에서 이미 `@Min(1)` 같은 annotation으로 검증한 request DTO 숫자 양수 여부를 application service에서 Kotlin `require`로 다시 검사하면 안 된다. 겉보기에는 안전해 보이지만 transport input concern이 use case로 이동하고 client-input failure에 `IllegalArgumentException`이 새어 나간다.

## 권장 패턴

inbound adapter가 request value shape와 primitive constraint를 검증한 뒤 command를 만든다. 실패는 project의 request validation exception shape로 변환하고, field source(`BODY`, `QUERY`, `PATH`, `HEADER`)와 field error를 포함한다.

application service는 이미 normalize된 command가 현재 service operation, state transition, tenant, feature flag, domain rule에서 허용되는지 같은 use-case policy에 집중한다. 이 실패에는 transport validation exception이 아니라 domain/application exception을 사용한다.

## 공통 원칙

같은 numeric/string constraint라도 HTTP request value가 syntactically valid한지에 관한 것이라면 adapter boundary에 속한다. valid value가 현재 business context에서 허용되는지에 관한 것이라면 service/domain layer에 속한다.

## 점검 방법

service method에서 inbound request class의 Bean Validation annotation으로 이미 표현된 constraint를 `require`, `check`, `IllegalArgumentException`으로 반복하는지 확인한다. common validation helper가 있다면 handler가 사용할 때 raw Bean Validation exception이 아니라 project의 request validation exception을 내보내는지도 확인한다.

## 검증 방법

invalid request value가 use case 호출 전에 400 validation response로 반환되는 handler/router test를 하나 둔다. application service가 primitive request-value check를 반복하지 않는다는 service test나 code review assertion도 둔다. project에 `awaitBodyValidated`나 query binding helper가 있다면 common validation helper test를 추가한다.
