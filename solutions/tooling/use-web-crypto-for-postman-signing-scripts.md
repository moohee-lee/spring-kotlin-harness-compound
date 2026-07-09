---
title: Postman signing script는 Web Crypto를 사용한다
tags: [postman, signature, webcrypto, harness]
scope: cross-project
status: active
principles: [build-tooling, testing-contracts]
---

# Postman signing script는 Web Crypto를 사용한다

> 관련 공통 원칙: [빌드와 도구 체인은 런타임을 오염시키지 않는다](../../principles/ko/build-tooling.md), [테스트는 계약을 검증해야 한다](../../principles/ko/testing-contracts.md)

## 적용 시점

Postman pre-request script가 HMAC request signature를 생성해야 하고, request가 전송되기 전에 현재 Postman sandbox가 지원하는 crypto API로 header를 계산해야 할 때 적용한다.

## 피해야 할 방향

오래된 snippet에서 복사한 `crypto-js` 예제에 의존하면 안 된다. 현재 Postman 문서는 `crypto-js`를 deprecated로 표시하고 Web Crypto object 사용을 안내한다.

## 권장 패턴

`crypto.subtle.importKey`와 `crypto.subtle.sign`으로 HMAC signing을 구현하고, header를 `pm.request.headers.upsert` 하기 전에 `await`로 계산 완료를 보장한다. signing 전에 `pm.variables.replaceIn(pm.request.url.toString())`으로 URL 안의 Postman variable을 실제 값으로 치환한 뒤 canonicalize한다.

## 공통 원칙

Postman signing script도 작은 platform-specific port로 다룬다. sandbox가 지원하는 crypto API를 사용하고, `upsert`로 duplicate header를 피하며, backend/reference signer의 canonicalization rule을 그대로 유지한다.

## 점검 방법

pre-request script에서 다음 신호를 찾는다.

- `crypto-js` import나 암묵적 사용
- `await` 없이 async signature를 만들고 header를 설정하는 코드
- `headers.add`로 `Authorization` header를 중복 추가하는 코드
- `{{variable}}` placeholder가 남은 URL을 그대로 signing하는 코드

## 검증 방법

reference implementation과 같은 fixed credential, timestamp, method, URL로 signing function을 실행한다. raw path percent-encoding case를 포함해 생성된 `Authorization` header와 date header가 golden vector와 일치해야 한다.
