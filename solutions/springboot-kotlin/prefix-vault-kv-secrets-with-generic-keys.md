---
title: Vault KV 비밀 값은 접두사로 네임스페이스를 만든다
tags: [springboot-kotlin, vault, configuration, secrets]
scope: cross-project
status: active
principles: [configuration-ownership]
---

# Vault KV 비밀 값은 접두사로 네임스페이스를 만든다

> 관련 공통 원칙: [설정 소유권과 로딩 시점](../../principles/ko/configuration-ownership.md)

## 적용 시점

Spring Cloud Vault ConfigData로 여러 KV path를 property source로 import하고, Vault 저장소가 `USER`, `PASSWORD` 같은 generic key name을 사용할 때 적용한다. application configuration은 database, Kafka, API signature 같은 credential을 구분해야 한다.

## 피해야 할 방향

`${USER}`, `${PASSWORD}`처럼 generic Vault key를 직접 참조하면 안 된다. 이 이름들은 OS environment variable, local shell variable, 다른 Vault path의 key와 충돌할 수 있고 Spring placeholder resolution이 의도하지 않은 값을 선택할 수 있다.

## 권장 패턴

contextual Vault location을 명시적 prefix와 full Vault KV mount path로 import한다.

```yaml
spring:
  config:
    import:
      - vault://secret/database/path?prefix=db.
      - vault://secret/kafka/path?prefix=kafka.
```

application config에서는 `${db.USER}`, `${db.PASSWORD}`, `${kafka.TRAFFIC_USERNAME}`, `${kafka.TRAFFIC_PASSWORD}`처럼 prefixed key를 참조한다.

Vault key name을 다른 팀이 관리하거나 이미 배포했다면 secret rename부터 요구하지 않는다. ConfigData import prefix로 application-local namespace를 만들고 YAML placeholder를 prefixed property name으로 유지한다.

명시적 `spring.config.import` location에서는 `spring.cloud.vault.kv.backend`가 contextual path 앞에 자동으로 붙는다고 가정하지 않는다. Spring Cloud Vault는 location path 자체를 읽는다. mount가 `secret`이고 secret path가 `svc-dev/nfv-dev/postgresql`라면 `vault://secret/svc-dev/nfv-dev/postgresql?prefix=db.`로 import해야 한다.

## 공통 원칙

외부 secret store의 key name은 application property namespace와 다르다. import 시점에 namespace를 부여해 runtime binding이 어느 secret source에서 온 값인지 드러나게 해야 한다.

## 점검 방법

`${USER}`, `${PASSWORD}`, `${TOKEN}`, `${SECRET}` 같은 generic placeholder를 검색한다. 여러 Vault context가 같은 key를 import하는지도 확인한다. runtime log에 `${db.USER}` 같은 literal placeholder가 datasource나 client library까지 도달했다면 각 `spring.config.import` location이 mount name을 포함한 full Vault path와 일치하는지 비교한다.

## 검증 방법

대표 Vault-prefixed property source로 application YAML을 load하는 configuration test를 만들고 최종 bound configuration이 expected value를 쓰는지 assert한다. Kafka dotted client property는 Spring Boot `Binder`로 `spring.kafka`를 bind해 `KafkaProperties` map에 expected JAAS string이 들어가는지 확인한다.
