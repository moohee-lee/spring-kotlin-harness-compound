---
title: Install C++ Runtime For Kafka Streams RocksDB JNI Images
tags: [springboot-kotlin, kafka-streams, docker, rocksdb]
scope: cross-project
status: active
---

# Install C++ Runtime For Kafka Streams RocksDB JNI Images

## Context
Spring Boot services that use Kafka Streams with state stores or `GlobalKTable`
load RocksDB through the `rocksdbjni` native library. Minimal runtime images,
especially Alpine-like images, may include a JRE but omit the C++ runtime
library that RocksDB needs.

## Wrong Direction
Do not treat `UnsatisfiedLinkError` from `/tmp/librocksdbjni*.so` as a Spring
Kafka configuration problem when the message says `libstdc++.so.6` is missing.
Changing Streams topology, state-store paths, or Kafka credentials does not
address a missing native runtime library in the container image.

## Correct Pattern
Install the C++ runtime in the image layer that runs the Spring Boot jar. For
Alpine-based images this is typically:

```dockerfile
RUN apk add --no-cache libstdc++
```

For Debian/Ubuntu-based images this is typically:

```dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends libstdc++6 \
    && rm -rf /var/lib/apt/lists/*
```

If the base image family is not stable or is hidden behind an internal tag, use
a guarded install that first checks for `libstdc++.so.6`, then tries `apk`, then
tries `apt-get`, and fails the image build with a clear message if neither is
available.

## Reusable Insight
Kafka Streams can start far enough for Spring Boot to create the application
context and then fail while the global stream thread initializes RocksDB. The
error belongs to the container runtime contract: any image that ships a Kafka
Streams app using RocksDB-backed stores must include the native C++ runtime.

## Detection
Look for stack traces containing:

- `Failed to start bean 'defaultKafkaStreamsBuilder'`
- `Exception caught during initialization of GlobalStreamThread`
- `UnsatisfiedLinkError: /tmp/librocksdbjni*.so`
- `Error loading shared library libstdc++.so.6`

Then inspect the Dockerfile or base image for an explicit `libstdc++` or
`libstdc++6` runtime install.

## Verification
Add a lightweight regression test or Dockerfile check that asserts the runtime
image installs `libstdc++`, and run the project tests. When registry access is
available, build the image and run a smoke command inside it:

```bash
docker buildx build --platform linux/amd64 -t <service>:rocksdb-check --load .
docker run --rm <service>:rocksdb-check sh -c "find /usr /lib -name 'libstdc++.so.6*' -print -quit"
```

For end-to-end verification, deploy the rebuilt image and confirm Kafka Streams
passes `GlobalStreamThread` initialization without a RocksDB JNI load error.
