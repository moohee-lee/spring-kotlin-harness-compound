---
title: RocksDB JNI 이미지에는 C++ 런타임을 설치한다
tags: [springboot-kotlin, kafka-streams, docker, rocksdb]
scope: cross-project
status: active
principles: [kafka-streams-operations, runtime-boundaries]
---

# RocksDB JNI 이미지에는 C++ 런타임을 설치한다

> 관련 공통 원칙: [Kafka Streams 운영 계약](../../principles/ko/kafka-streams-operations.md), [런타임 경계와 컨테이너 계약](../../principles/ko/runtime-boundaries.md)

## 적용 시점

Kafka Streams state store나 `GlobalKTable`을 사용하는 Spring Boot service는 `rocksdbjni` native library를 통해 RocksDB를 로드한다. Alpine 계열처럼 최소화된 runtime image는 JRE는 포함하지만 RocksDB가 요구하는 C++ runtime library를 생략할 수 있다.

## 피해야 할 방향

`/tmp/librocksdbjni*.so`에서 발생한 `UnsatisfiedLinkError`가 `libstdc++.so.6` 누락을 말하고 있다면 Spring Kafka 설정 문제로 다루면 안 된다. Streams topology, state-store path, Kafka credential을 바꿔도 container image 안의 native runtime library 부재는 해결되지 않는다.

## 권장 패턴

Spring Boot jar를 실행하는 image layer에 C++ runtime을 설치한다. Alpine 기반 image라면 보통 다음과 같다.

```dockerfile
RUN apk add --no-cache libstdc++
```

Debian/Ubuntu 기반 image라면 보통 다음과 같다.

```dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends libstdc++6 \
    && rm -rf /var/lib/apt/lists/*
```

base image family가 불안정하거나 internal tag 뒤에 숨겨져 있다면 먼저 `libstdc++.so.6` 존재 여부를 확인하고, 없으면 `apk`, 그다음 `apt-get`을 시도한 뒤 둘 다 불가능하면 명확한 message로 image build를 실패시키는 guarded install을 둔다.

## 공통 원칙

Kafka Streams는 Spring Boot application context 생성 이후 global stream thread가 RocksDB를 초기화하는 단계에서 실패할 수 있다. 이 오류는 container runtime contract에 속한다. RocksDB-backed store를 쓰는 Kafka Streams app image는 native C++ runtime을 포함해야 한다.

## 점검 방법

stack trace에서 다음 신호를 찾는다.

- `Failed to start bean 'defaultKafkaStreamsBuilder'`
- `Exception caught during initialization of GlobalStreamThread`
- `UnsatisfiedLinkError: /tmp/librocksdbjni*.so`
- `Error loading shared library libstdc++.so.6`

이후 Dockerfile이나 base image에 `libstdc++` 또는 `libstdc++6` runtime install이 있는지 확인한다.

## 검증 방법

가능하면 Dockerfile check나 lightweight regression test로 runtime image가 `libstdc++`를 설치하는지 확인한다. registry 접근이 가능하면 image를 build한 뒤 smoke command를 실행한다.

```bash
docker buildx build --platform linux/amd64 -t <service>:rocksdb-check --load .
docker run --rm <service>:rocksdb-check sh -c "find /usr /lib -name 'libstdc++.so.6*' -print -quit"
```

end-to-end 검증에서는 rebuilt image를 배포하고 Kafka Streams가 RocksDB JNI load error 없이 `GlobalStreamThread` initialization을 통과하는지 확인한다.
