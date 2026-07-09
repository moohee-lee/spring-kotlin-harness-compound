---
title: Coroutine Spring Boot 서비스의 기본 관측성은 OpenTelemetry Java agent로 둔다
tags: [springboot-kotlin, opentelemetry, coroutines, observability]
scope: cross-project
status: active
principles: [adapter-boundaries, runtime-boundaries]
---

# Coroutine Spring Boot 서비스의 기본 관측성은 OpenTelemetry Java agent로 둔다

> 관련 공통 원칙: [어댑터 경계와 재사용 구조](../../principles/ko/adapter-boundaries.md), [런타임 경계와 컨테이너 계약](../../principles/ko/runtime-boundaries.md)

## 적용 시점

Spring Boot Kotlin service가 WebFlux/Reactor, `suspend` application service, custom coroutine dispatcher의 `withContext`, JDBC/jOOQ, Kafka, Logback을 함께 사용할 때 적용한다. coroutine 때문에 OpenTelemetry를 위해 application code를 바꿔야 하는지 질문이 나올 수 있다.

## 피해야 할 방향

Kotlin coroutine을 쓴다는 이유만으로 application에 full OpenTelemetry SDK와 custom span lifecycle management를 먼저 추가하면 안 된다. Java agent를 켠 뒤에도 custom MDC `traceId` logging만 correlation key로 유지하면 안 된다. agent는 `trace_id`, `span_id` 같은 standard MDC key를 쓰고, custom request filter는 inbound `traceparent`가 없을 때 다른 trace id를 만들 수 있다.

## 권장 패턴

일반 Spring Boot JVM deployment에서는 OpenTelemetry Java agent를 기본값으로 둔다. agent는 Kotlin coroutine, Reactor, Java executor, WebFlux, Kafka, JDBC, HikariCP, Logback MDC를 포함해 broad automatic instrumentation을 제공한다. service가 manual span, custom attribute, Java agent 대신 starter-based configuration을 정말 필요로 할 때만 application OpenTelemetry dependency를 추가한다.

coroutine code 안에서 manual span이 필요하면 OpenTelemetry API와 Kotlin context extension을 추가하고, `withContext`, `async`, launched work 주변에서 `Context.current()`를 coroutine context element로 전파한다.

## 공통 원칙

Coroutine 사용은 기본 instrumentation 선택을 바꾸는 것이 아니라 context propagation 검증 대상을 바꾼다. Java agent를 쓰면 대부분의 propagation은 자동이다. project-specific 작업은 주로 Kubernetes agent injection/configuration과 log correlation cleanup이다.

## 점검 방법

custom `CoroutineDispatcher` bean, `withContext`, `async`, `runBlocking`을 검색한다. OpenTelemetry가 Java agent/operator로 들어오는지, application dependency로 들어오는지 확인한다. `logback-spring.xml`에서는 agent MDC injection 이후 custom `%X{traceId}` pattern을 `%X{trace_id}`, `%X{span_id}`와 맞춰야 하는지 확인한다.

## 검증 방법

Java agent와 local 또는 in-cluster Collector로 app을 실행한다. HTTP, outbound HTTP/Feign, Kafka, JDBC path를 호출한 뒤 server span, child client/database/messaging span, 동일 `trace_id`를 가진 log line이 한 trace로 이어지는지 확인한다. coroutine path에서는 `withContext` dispatcher와 `async` fan-out을 지나도 span parent 관계가 유지되는지 확인한다.
