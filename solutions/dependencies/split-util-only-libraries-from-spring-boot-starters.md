---
title: Utility-only library와 Spring Boot starter는 분리한다
tags: [springboot-kotlin, harness, dependencies, autoconfiguration]
scope: cross-project
status: active
principles: [adapter-boundaries]
---

# Utility-only library와 Spring Boot starter는 분리한다

> 관련 공통 원칙: [어댑터 경계와 재사용 구조](../../principles/ko/adapter-boundaries.md)

## 적용 시점

공유 회사 library에서 plain utility package 하나만 쓰고 싶은데, 같은 jar 안에 Spring Boot auto-configuration metadata, logging 설정, message bundle, property source, web interceptor, datasource 설정 같은 runtime side effect가 함께 들어 있을 때 적용한다.

## 피해야 할 방향

전체 공유 library를 runtime dependency로 추가한 뒤 "특정 package만 import했으니 괜찮다"고 판단하면 안 된다. Gradle exclude는 transitive module을 제거할 뿐, 같은 jar 안의 class나 resource 일부를 골라 제거하지 못한다. jar가 runtime classpath에 올라오면 Spring Boot는 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`를 발견할 수 있고, logging도 `logback-spring.xml` 같은 classpath resource를 읽을 수 있다.

## 권장 패턴

artifact를 분리한다.

- utility code, model, constant, exception만 담는 plain `java-library` 또는 Kotlin library artifact를 둔다. 이 artifact에는 Boot auto-configuration import, application logging file, 환경별 resource를 넣지 않는다.
- Spring Boot auto-configuration, logging integration, web adapter, message source, infrastructure bean은 별도 starter/autoconfigure artifact가 소유한다.

utility만 필요한 consumer는 util artifact만 의존하고, platform behavior 전체가 필요한 consumer만 starter를 의존한다.

## 공통 원칙

runtime 경계는 Java package import가 아니라 classpath다. jar 하나에 utility와 Boot integration metadata가 함께 있으면, class 하나만 사용해도 metadata와 resource가 함께 들어온다.

## 점검 방법

utility 목적으로 공유 dependency를 추가하기 전에 jar나 source에서 다음 항목을 확인한다.

- `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
- `META-INF/spring.factories`
- `logback.xml`, `logback-spring.xml`
- message/property resource
- broad `@AutoConfiguration` class

이 항목이 있으면 utility-only dependency가 아니라 starter-style dependency로 취급한다.

## 검증 방법

consumer service에서 `ApplicationContextRunner` 또는 focused boot startup test를 실행해 원치 않는 bean이나 auto-configuration이 생기지 않는지 확인한다. logging 초기화가 application 자신의 설정을 사용하는지도 확인한다. 장기적으로는 util artifact jar를 검사해 의도한 utility package/resource만 있고 Boot metadata가 없는지 검증한다.
