---
title: 외부 client test URL은 inert local URL을 사용한다
tags: [springboot-kotlin, configuration, testing, http-client]
scope: cross-project
status: active
principles: [configuration-ownership, testing-contracts]
---

# 외부 client test URL은 inert local URL을 사용한다

> 관련 공통 원칙: [설정 소유권과 로딩 시점](../../principles/ko/configuration-ownership.md), [테스트는 계약을 검증해야 한다](../../principles/ko/testing-contracts.md)

## 적용 시점

Spring Boot service가 Feign, WebClient, RestClient, Kafka schema registry 등 외부 adapter endpoint를 profile YAML로 설정할 때 적용한다. production-like profile은 실제 내부 URL이 필요할 수 있지만 test profile은 context startup과 focused unit test를 만족할 만큼의 설정만 필요하다.

## 피해야 할 방향

`application-test.yaml`에 현실적인 dev, stage, test environment hostname을 복사하면 안 된다. future context test, smoke test, 잘못 wiring된 client가 local test process 안에서 실패하지 않고 내부 service를 호출할 수 있다.

## 권장 패턴

실제 외부 endpoint는 `application-dev.yaml`이나 deployment-managed configuration 같은 owning runtime profile에 둔다. `application-test.yaml`에는 test가 local stub server를 직접 띄우고 해당 property를 그 server로 지정하는 경우를 제외하고 `http://localhost` 같은 inert local URL을 사용한다.

## 공통 원칙

Test configuration은 binding과 bean creation을 만족해야 하지만 실제 infrastructure 접근을 암시하면 안 된다. local placeholder URL은 accidental call을 시끄럽고 제한된 방식으로 실패하게 만들면서 production code가 bind하는 property path는 보존한다.

## 점검 방법

test resource에서 `dev`, `stage`, `internal`, cloud domain, service-discovery name처럼 실제 shared environment처럼 보이는 hostname을 검색한다. test profile에 그런 값이 있으면 어떤 test가 의도적으로 그 endpoint를 호출하는지 확인하고, 아니라면 inert local URL로 바꾼다.

## 검증 방법

base와 profile YAML을 분리해서 load하는 configuration test를 둔다. real endpoint 값은 runtime profile에만 있고 test profile은 local 또는 stub-owned endpoint를 쓰는지 assert한다. full Spring context test를 실행해 placeholder가 bean creation을 여전히 만족하는지도 확인한다.
