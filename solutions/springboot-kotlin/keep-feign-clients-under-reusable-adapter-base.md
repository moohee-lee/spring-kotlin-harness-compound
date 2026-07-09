---
title: Feign client는 재사용 가능한 adapter base 아래에 둔다
tags: [springboot-kotlin, feign, architecture, configuration]
scope: cross-project
status: active
principles: [adapter-boundaries]
---

# Feign client는 재사용 가능한 adapter base 아래에 둔다

> 관련 공통 원칙: [어댑터 경계와 재사용 구조](../../principles/ko/adapter-boundaries.md)

## 적용 시점

Spring Boot Kotlin service가 하나의 gateway용 Feign client로 시작했다가 시간이 지나며 client가 늘어날 수 있을 때 적용한다. 초기에 정한 package와 bean name은 이후 contributor가 복사하는 기본 pattern이 된다.

## 피해야 할 방향

`adapter.output.feign.identity`처럼 특정 client package만 scan하거나, shared infrastructure에 `identityFeignCoroutineDispatcher` 같은 domain-specific 이름을 붙이면 안 된다. 이후 다른 client가 추가될 때 scan annotation, dispatcher bean, 실제로는 generic한 configuration이 중복된다.

## 권장 패턴

`@EnableFeignClients`는 `adapter.output.feign` 같은 reusable base package를 scan하게 한다. scan annotation은 application class가 아니라 `FeignClientConfiguration` 같은 shared Feign infrastructure configuration에 둔다. client는 `adapter.output.feign.{domain}` 아래에 두고, shared Feign infrastructure는 `adapter.output.feign.config` 또는 `adapter.output.feign.support` 아래에 둔다. timeout, retry, pool, circuit breaker, protocol policy가 실제로 다를 때만 client-specific configuration을 만든다.

signed Feign call은 `SignatureFeignConfiguration` 같은 reusable opt-in configuration을 shared Feign config package에 둔다. 이 configuration이나 interceptor를 global `@Component`/`@Configuration`으로 등록하지 말고, signature가 필요한 Feign client에서만 `@FeignClient(configuration = [...])`로 붙인다.

Feign interface annotation을 reflection으로 읽어 runtime URL을 재구성하지 않는다. `@FeignClient(url = "\${...}")`는 placeholder이고 effective target은 Spring Cloud OpenFeign lifecycle에서 resolve된다. interceptor 안에서는 Feign이 resolve한 `RequestTemplate`에서 HTTP method와 final target URL을 얻는다.

pure signing function이 이미 있다면 credential `ConfigurationProperties`는 signed HTTP adapter 근처, 예를 들어 Feign config package의 `SignatureSecretProperties`로 둔다. opt-in Feign configuration에 주입해서 pure function을 직접 호출한다. property를 같은 function에 넘기기만 하는 Spring service는 policy를 추가하지 않으면서 bean과 test surface만 늘린다.

## 공통 원칙

client adapter infrastructure는 처음 등장한 domain 이름에 갇히지 않아야 한다. shared scan, dispatcher, signing support는 common adapter base에 두고, client-specific policy만 opt-in으로 분리한다.

## 점검 방법

다음을 검색한다.

- 특정 domain package에서 끝나는 `@EnableFeignClients(basePackages = [...])`
- application class에 남아 있는 Feign scan annotation
- generic infrastructure인데 gateway 이름이 들어간 bean name
- domain package마다 복제된 signed-client configuration
- stereotype annotation으로 global 등록된 signature interceptor
- property를 pure signer에 전달하기만 하는 service

## 검증 방법

shared Feign configuration이 common Feign base package를 scan하고 application class가 Feign scanning을 소유하지 않는지 focused test로 확인한다. blocking Feign call이 shared virtual-thread dispatcher에서 실행되는지, signature interceptor가 global `@Component`가 아니라 Feign client's `configuration`을 통해 opt-in되는지도 확인한다. signed client는 interceptor가 method와 URL을 Feign의 resolved `RequestTemplate`에서 얻는지 test한다. package move 뒤 detekt와 full test suite를 실행한다.
