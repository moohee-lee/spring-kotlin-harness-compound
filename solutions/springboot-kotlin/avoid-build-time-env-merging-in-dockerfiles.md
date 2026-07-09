---
title: Dockerfile에서 build-time ENV merge를 하지 않는다
tags: [springboot-kotlin, docker, jvm-options]
scope: cross-project
status: active
principles: [runtime-boundaries, build-tooling]
---

# Dockerfile에서 build-time ENV merge를 하지 않는다

> 관련 공통 원칙: [런타임 경계와 컨테이너 계약](../../principles/ko/runtime-boundaries.md), [빌드와 도구 체인은 런타임을 오염시키지 않는다](../../principles/ko/build-tooling.md)

## 적용 시점

Spring Boot JVM service가 Dockerfile entrypoint에서 `JAVA_OPTS`, `JAVA_OPTIONS`, `JVM_OPTIONS` 같은 runtime 환경 변수를 받아 JVM option을 조합할 때 적용한다.

## 피해야 할 방향

Dockerfile `ENV`에서 runtime 환경 변수를 merge하려고 하면 안 된다.

```Dockerfile
ENV JAVA_OPTIONS="${JAVA_OPTS} ${JAVA_OPTIONS}"
```

Docker는 이 표현을 image build time에 평가한다. 나중에 `docker run -e`나 Kubernetes env로 들어오는 값은 여기서 merge되지 않는다. Docker static checker도 Dockerfile 안에서 미리 정의되지 않은 변수에 대해 `UndefinedVar`를 보고한다.

## 권장 패턴

`ENV`에는 안정적인 image default만 두고, runtime 변수 조합은 shell-form entrypoint에서 수행한다.

```Dockerfile
ENV JAVA_OPTIONS=""
ENV JVM_OPTIONS="-XshowSettings:vm -XX:MaxRAMPercentage=60.0 -XX:+UseG1GC"

ENTRYPOINT ["sh","-c","exec java $JAVA_OPTS $JAVA_OPTIONS $JVM_OPTIONS -jar app.jar"]
```

이 방식은 container runtime이 `JVM_OPTIONS`를 override하거나 `JAVA_OPTS`, `JAVA_OPTIONS`를 공급해도 build-time interpolation에 의존하지 않는다.

## 공통 원칙

Dockerfile `ENV`는 image build metadata이지 runtime expression engine이 아니다. runtime JVM flag 조합은 runtime shell이 변수를 확장하는 지점에서 수행해야 한다.

## 점검 방법

Dockerfile static check를 실행하고 `ENV` instruction이 `JAVA_OPTS`, `JAVA_OPTIONS` 같은 runtime-only 변수를 참조하면서 `UndefinedVar` warning을 내는지 확인한다.

## 검증 방법

```bash
docker buildx build --check .
```

Apple Silicon local machine에서 amd64-only base image를 쓰면 builder platform 설정에 따라 별도의 `InvalidBaseImagePlatform` warning이 남을 수 있다. 이 경우에도 JVM option 처리에 대한 `UndefinedVar` warning은 없어야 한다.
