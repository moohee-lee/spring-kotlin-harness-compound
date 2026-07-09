---
title: Skeleton overlay에서 Kotlin을 override하면 detekt 버전도 맞춘다
tags: [springboot-kotlin, harness, detekt, kotlin, initializr]
scope: cross-project
status: active
principles: [build-tooling]
---

# Skeleton overlay에서 Kotlin을 override하면 detekt 버전도 맞춘다

> 관련 공통 원칙: [빌드와 도구 체인은 런타임을 오염시키지 않는다](../../principles/ko/build-tooling.md)

## 적용 시점

Spring Boot Kotlin Initializr scaffold가 사용자가 요청한 Kotlin version을 보존하면서 reusable skeleton overlay를 적용하고, 그 overlay가 detekt 설정도 함께 가져올 때 적용한다.

## 피해야 할 방향

Skeleton의 `PluginVersions.DETEKT`만 복사하거나 `PluginVersions.KOTLIN`만 보존하면 compiler mismatch가 생길 수 있다. 예를 들어 detekt `2.0.0-alpha.2`가 Kotlin `2.3.0`으로 compile되어 있는데 target project가 Kotlin `2.4.0`을 의도적으로 쓰면, detekt는 code issue를 보고하기 전에 version mismatch로 실패한다.

## 권장 패턴

detekt를 tool-specific compatibility dependency로 다룬다.

- 사용자가 요청한 Kotlin version은 유지한다.
- detekt 공식 compatibility table을 확인한다.
- `PluginVersions.DETEKT`와 `dev.detekt:detekt-rules-ktlint-wrapper` 같은 `detektPlugins(...)` artifact를 Kotlin version과 맞는 detekt version으로 함께 올린다.

Kotlin `2.4.0`을 쓰는 경우라면 compatibility table에서 맞는 detekt version을 확인하고, Kotlin을 낮추는 대신 detekt 쪽을 맞춘다.

## 공통 원칙

Skeleton overlay가 target project의 version을 보존하는 것만으로는 충분하지 않다. Kotlin compiler를 내장하거나 실행하는 build tool은 target Kotlin version 변경에 맞춰 별도 compatibility 정렬이 필요하다.

## 점검 방법

Overlay 적용 후 `./gradlew detekt --stacktrace`를 실행한다. mismatch는 보통 `detekt was compiled with Kotlin <x> but is currently running with <y>` 형태로 드러난다. `PluginVersions.KOTLIN`이 skeleton source와 다르면 `buildSrc/src/main/kotlin/PluginVersions.kt`와 detekt compatibility table을 함께 비교한다.

## 검증 방법

```bash
env GRADLE_USER_HOME=.gradle-user-home ./gradlew detekt --no-daemon --stacktrace
env GRADLE_USER_HOME=.gradle-user-home ./gradlew build --no-daemon --stacktrace
```

두 명령이 detekt Kotlin compiler mismatch 없이 통과해야 한다. detekt가 이후 일반 style issue를 보고하면, 그것은 tool-version 정렬과 분리해서 source/config 문제로 처리한다.
