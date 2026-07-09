---
title: Porting한 signing utility는 외부 golden vector로 검증한다
tags: [springboot-kotlin, harness, testing, signature]
scope: cross-project
status: active
principles: [testing-contracts]
---

# Porting한 signing utility는 외부 golden vector로 검증한다

> 관련 공통 원칙: [테스트는 계약을 검증해야 한다](../../principles/ko/testing-contracts.md)

## 적용 시점

service가 다른 repository의 signing, hashing, encryption, canonicalization, header-building utility를 dependency로 쓰지 않고 porting할 때 적용한다. HMAC signature는 canonical host, path, timestamp, scope, percent encoding 차이만으로도 겉보기에는 valid하지만 실제로는 거부되는 signature가 만들어질 수 있어 특히 중요하다.

## 피해야 할 방향

Spring wrapper test가 자신이 delegate하는 local helper output과 비교하는 것만으로는 충분하지 않다. 그것은 delegation/config binding test일 뿐이다. helper가 틀리면 actual과 expected가 같은 방식으로 틀려도 test는 통과한다. hard-coded signature도 새로 porting한 implementation으로 생성한 값이라면 독립성이 약하다.

## 권장 패턴

두 층의 test를 둔다.

- wrapper test는 configuration property가 signing helper로 전달되는지 확인한다.
- contract test는 reference implementation이나 protocol document에서 생성한 golden vector로 canonical string-to-sign과 final header를 검증한다.

URL convenience API가 있다면 boundary conversion도 명시적으로 테스트한다. URL 또는 HTTP client request에서 host, raw path, optional canonicalization, scope, emitted header로 어떻게 변환되는지 확인한다.

## 공통 원칙

ported crypto/signature utility에서 `expected = localHelper(...)`는 tautology다. expected value는 independent oracle에서 와야 하며, 가능하면 intermediate canonical data를 test에 노출해야 한다.

## 점검 방법

expected value를 production helper가 다시 만드는 test, service test가 local call과 data-class equality만 assert하는 test를 찾는다. string-to-sign, scope header presence, raw path percent-encoding, empty/root path handling, query exclusion에 대한 assertion이 빠져 있는지도 확인한다.

## 검증 방법

fixed credential, fixed clock, representative request input으로 reference implementation을 실행한다. golden string-to-sign, signature hex, authorization header, timestamp, optional scope header를 test에 기록한다. ported implementation을 같은 vector에 대해 실행하고, canonicalization step을 일부러 바꾸면 focused test가 실패해야 한다.
