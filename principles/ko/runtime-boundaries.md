# 런타임 경계와 컨테이너 계약

## 원칙

이미지 build time, container runtime, Kubernetes lifecycle, Spring lifecycle은 서로 다른 경계다. 한 경계에서만 가능한 일을 다른 경계로 밀어 넣으면 설정이 적용되지 않거나 종료, native library, state 복구가 깨진다.

## 적용 기준

- Dockerfile `ENV`는 build metadata이며 runtime expression engine이 아니다.
- 런타임 JVM option merge는 entrypoint shell에서 수행한다.
- runtime image에는 애플리케이션 native dependency가 실제로 필요한 shared library를 포함한다.
- Kubernetes termination grace는 Spring shutdown phase, Kafka Streams close timeout, `preStop` delay보다 길어야 한다.
- Docker build context는 Dockerfile이 실제로 읽는 입력만 포함한다.

## 피해야 할 신호

- Dockerfile `ENV`에서 runtime-only env var를 참조한다.
- `preStop` sleep만 추가하고 `terminationGracePeriodSeconds`는 그대로 둔다.
- runtime image에서 `libstdc++.so.6` 같은 native dependency가 없는데 Spring 설정을 먼저 의심한다.
- jar-only image build에 전체 repository와 Gradle cache가 build context로 올라간다.

## 검증 방법

- `docker buildx build --check .`로 Dockerfile static warning을 확인한다.
- image smoke command로 native library 존재 여부를 확인한다.
- `helm template`로 termination grace와 lifecycle hook을 확인한다.
- BuildKit의 `load build context` 크기를 확인한다.
