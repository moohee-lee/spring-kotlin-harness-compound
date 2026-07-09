---
title: 테스트에서는 Vault ConfigData import를 프로필로 차단한다
tags: [springboot-kotlin, vault, configdata, testing]
scope: cross-project
status: active
principles: [configuration-ownership, testing-contracts]
---

# 테스트에서는 Vault ConfigData import를 프로필로 차단한다

> 관련 공통 원칙: [설정 소유권과 로딩 시점](../../principles/ko/configuration-ownership.md), [테스트는 계약을 검증해야 한다](../../principles/ko/testing-contracts.md)

## 적용 시점

Spring Cloud Vault를 `spring.config.import`로 추가한 service에 `@SpringBootTest`나 integration test가 있고, test profile은 Testcontainers-backed datasource 등 test-owned 설정만 사용해야 할 때 적용한다. 테스트는 live Vault 접근에 의존하면 안 된다.

## 피해야 할 방향

base application document에 `spring.config.import: vault://...`를 두고 `application-test.yaml`에만 `spring.cloud.vault.enabled=false`를 추가하는 방식에 의존하면 안 된다. Vault ConfigData processing은 test profile이 Vault를 비활성화하기 전에 실행될 수 있다. developer machine에 valid credential이 있어서 테스트가 통과하더라도 의도하지 않은 network call을 수행했을 수 있다.

## 권장 패턴

profile file을 사용하는 project라면 Vault ConfigData import와 Vault authentication setting을 `application-dev.yaml`, `application-local.yaml` 같은 profile-specific file에 둔다.

```yaml
spring:
  config:
    import:
      - vault:///service/database?prefix=db.
```

single `application.yaml`을 써야 한다면 import를 profile-gated YAML document에 둔다.

```yaml
---
spring:
  config:
    activate:
      on-profile: "!test"
    import:
      - vault:///service/database?prefix=db.
```

`application-test.yaml`의 `spring.cloud.vault.enabled: false`는 명시적 guard로 유지하되, 유일한 isolation mechanism으로 의존하지 않는다.

## 공통 원칙

ConfigData import는 early environment construction의 일부다. test isolation은 나중에 Vault property가 disabled라고 말하는 것보다 import location 자체가 test profile에서 inactive일 때 더 안정적이다.

## 점검 방법

Vault import 추가 후 full Spring context test를 실행하고 log에서 Vault HTTP call을 찾는다. login, secret read, `auth/token/revoke-self` 같은 call이 보이면 통과한 테스트도 잘못된 것이다.

## 검증 방법

`test` profile로 full test suite를 실행하고 output에 Vault HTTP call이 없는지 확인한다. configuration test로 base `application.yaml`에 Vault import가 없고, profile-specific application file에 Vault import가 있으며, `application-test.yaml`에 `spring.cloud.vault.enabled: false`가 있는지 assert한다.
