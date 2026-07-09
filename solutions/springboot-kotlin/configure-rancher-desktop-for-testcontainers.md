---
title: Rancher Desktop 환경에서는 Testcontainers discovery 경로를 명시한다
tags: [springboot-kotlin, testcontainers, rancher-desktop, jooq, gradle]
scope: cross-project
status: active
principles: [testing-contracts, build-tooling]
---

# Rancher Desktop 환경에서는 Testcontainers discovery 경로를 명시한다

> 관련 공통 원칙: [테스트는 계약을 검증해야 한다](../../principles/ko/testing-contracts.md), [빌드와 도구 체인은 런타임을 오염시키지 않는다](../../principles/ko/build-tooling.md)

## 적용 시점

macOS 개발 환경에서 Rancher Desktop을 Docker runtime으로 사용하고, Gradle task, test, jOOQ code generation이 Testcontainers를 사용할 때 적용한다. Docker CLI는 `rancher-desktop` context로 정상 동작하지만 Testcontainers는 Docker environment를 찾지 못할 수 있다.

## 피해야 할 방향

`docker info`가 성공한다고 해서 Testcontainers도 같은 Docker CLI context를 자동으로 상속한다고 가정하면 안 된다. Testcontainers는 environment variable과 well-known socket path를 통해 Docker를 찾는다. Rancher Desktop에서는 host socket이 `/var/run/docker.sock`이 아니라 `$HOME/.rd/docker.sock`인 경우가 많다.

## 권장 패턴

Rancher Desktop이 moby/dockerd engine을 사용할 때는 Gradle 실행 전에 Testcontainers용 환경 변수를 명시한다.

```bash
export DOCKER_HOST="unix://$HOME/.rd/docker.sock"
export TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE=/var/run/docker.sock
export TESTCONTAINERS_HOST_OVERRIDE=$(rdctl shell ip a show vznat | awk '/inet / {sub("/.*",""); print $2}')
```

설치된 Rancher Desktop version이 `rdctl info --field ip-address`를 제공한다면 그 값을 `TESTCONTAINERS_HOST_OVERRIDE`로 사용할 수도 있다. Testcontainers-backed Gradle task를 자주 실행하는 project라면 shell profile에 저장한다.

## 공통 원칙

Docker CLI context와 Testcontainers Docker discovery는 서로 다른 경로다. `docker context ls`와 `docker info`가 정상이어도 project는 "Could not find a valid Docker environment"로 실패할 수 있다.

## 점검 방법

다음 경계를 각각 확인한다.

```bash
docker context ls
docker info
printenv DOCKER_HOST
ls -l /var/run/docker.sock "$HOME/.rd/docker.sock"
```

`docker info`는 `rancher-desktop`으로 동작하고, `DOCKER_HOST`는 비어 있으며, `/var/run/docker.sock`이 없으면 Testcontainers가 daemon을 놓칠 가능성이 높다. `DOCKER_HOST`만 설정했을 때 `$HOME/.rd/docker.sock` 관련 Ryuk mount error로 바뀐다면 `TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE`도 추가한다.

## 검증 방법

원래 실패하던 Testcontainers-backed command를 같은 환경 변수로 실행한다.

```bash
DOCKER_HOST="unix://$HOME/.rd/docker.sock" \
TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE=/var/run/docker.sock \
TESTCONTAINERS_HOST_OVERRIDE="$(rdctl shell ip a show vznat | awk '/inet / {sub("/.*",""); print $2}')" \
./gradlew jooqCodegen
```

성공하면 문제는 jOOQ generator 설정이 아니라 Docker discovery였다.
