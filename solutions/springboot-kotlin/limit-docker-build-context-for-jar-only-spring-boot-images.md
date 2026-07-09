---
title: jar-only Spring Boot image는 Docker build context를 제한한다
tags: [springboot-kotlin, docker, github-actions]
scope: cross-project
status: active
principles: [runtime-boundaries, build-tooling]
---

# jar-only Spring Boot image는 Docker build context를 제한한다

> 관련 공통 원칙: [런타임 경계와 컨테이너 계약](../../principles/ko/runtime-boundaries.md), [빌드와 도구 체인은 런타임을 오염시키지 않는다](../../principles/ko/build-tooling.md)

## 적용 시점

Gradle로 jar를 빌드한 뒤 runtime-only Dockerfile로 Spring Boot image를 만들고, Dockerfile이 사실상 `Dockerfile`과 `build/libs/*.jar`만 필요로 할 때 적용한다.

## 피해야 할 방향

Dockerfile이 built jar만 복사하는데 repository에 `.dockerignore`가 없으면 안 된다. GitHub Actions에서 Gradle build 후 Docker build를 실행할 때 `.git`, `.gradle`, generated source, test report, compiled class, 기타 build output이 image에는 필요 없는데도 BuildKit context로 업로드된다.

## 권장 패턴

deny-by-default `.dockerignore`를 두고 runtime image 입력만 허용한다.

```dockerignore
**
!Dockerfile
!build/
!build/libs/
!build/libs/*.jar
```

Docker build 전에 `./gradlew clean bootJar` 같은 jar task를 먼저 실행해 allowlist된 jar가 BuildKit context를 읽는 시점에 존재하게 한다.

## 공통 원칙

jar-only Spring Boot Dockerfile에서 build context 크기는 CI 계약이다. Dockerfile input set이 작고 명확하므로 ignore file이 그 경계를 드러내야 한다.

## 점검 방법

local에서 `docker buildx build ... .`를 실행하고 `load build context` line을 확인한다. Dockerfile이 `COPY ./build/libs/*.jar` 정도만 수행하는데 context가 수십 MB나 수백 MB라면 focused `.dockerignore`가 없는 것이다.

## 검증 방법

```bash
./gradlew clean bootJar
docker buildx build --platform linux/amd64 -t <service>:workflow-check --load .
```

Docker build는 여전히 `build/libs/*.jar`를 복사해야 하고, BuildKit log는 전체 repository와 Gradle build tree가 아니라 작은 context 전송량을 보여야 한다.
