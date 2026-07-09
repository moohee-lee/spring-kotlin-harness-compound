---
title: Spring Boot virtual thread는 application YAML로 활성화한다
tags: [springboot-kotlin, spring-boot, virtual-threads]
scope: cross-project
status: active
principles: [build-tooling, runtime-boundaries]
---

# Spring Boot virtual thread는 application YAML로 활성화한다

> 관련 공통 원칙: [빌드와 도구 체인은 런타임을 오염시키지 않는다](../../principles/ko/build-tooling.md), [런타임 경계와 컨테이너 계약](../../principles/ko/runtime-boundaries.md)

## 적용 시점

Java 21+에서 실행되는 Spring Boot Kotlin service가 Spring Boot auto-configured application task execution에 virtual thread를 사용하게 하려는 경우 적용한다.

## 피해야 할 방향

몇몇 adapter에서만 `Executors.newVirtualThreadPerTaskExecutor()`를 직접 만드는 것으로는 Spring Boot의 global virtual-thread mode가 켜지지 않는다. Boot-managed task execution은 여전히 default platform-thread 설정으로 남는다.

## 권장 패턴

`application.yaml`에 Boot property를 설정한다.

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

blocking JDBC나 thread-bound integration 때문에 명시적으로 scoped virtual-thread dispatcher가 필요한 곳은 그대로 유지한다. Boot property는 그런 targeted executor를 대체하기보다 framework-managed execution을 보완한다.

## 공통 원칙

Spring Boot 3.2+와 4.x에서 `spring.threads.virtual.enabled`는 framework-level switch다. WebFlux application에서 이 설정이 Netty event loop를 virtual thread로 바꾸지는 않는다. nonblocking event loop는 event loop로 남고, Boot-managed task execution에 적용된다.

## 점검 방법

`application.yaml`과 profile-specific config에서 `spring.threads.virtual.enabled`를 검색한다. production code의 custom virtual-thread executor도 함께 검색해 어떤 blocking work가 별도 execution boundary를 갖는지 확인한다.

## 검증 방법

YAML binding test로 `spring.threads.virtual.enabled`가 `true`인지 확인하고, Spring context load test로 해당 설정에서 애플리케이션이 시작되는지 검증한다. 전체 integration test는 Docker/Testcontainers 같은 외부 서비스 조건에 여전히 의존할 수 있다.
