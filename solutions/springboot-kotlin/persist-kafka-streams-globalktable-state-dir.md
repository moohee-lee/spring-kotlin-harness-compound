---
title: Persist Kafka Streams GlobalKTable State Dir
tags: [springboot-kotlin, kafka-streams, globalktable, state-dir]
scope: cross-project
status: active
---

# Persist Kafka Streams GlobalKTable State Dir

## Context
Spring Boot services using Kafka Streams `GlobalKTable` joins can appear to
replay the full table topic on every restart when the Streams local state
directory is under a temporary container path, gets cleaned on startup, or is not
mounted as persistent storage. Kafka Streams already persists `GlobalKTable`
state and checkpoint offsets locally when the `state.dir` survives restarts.

## Wrong Direction
Do not add a separate application-managed disk cache or offset table before
checking the Kafka Streams state directory. Duplicating state outside Kafka
Streams risks offset/store inconsistency and adds recovery code that the Streams
runtime already provides.

## Correct Pattern
Configure a stable Kafka Streams state directory through Spring Boot:

```yaml
spring:
  kafka:
    streams:
      state-dir: /var/lib/kcp-network-consumer/kafka-streams
      cleanup:
        on-startup: false
        on-shutdown: false
```

Mount that path to a persistent volume in containerized environments. Keep
`spring.kafka.streams.application-id` stable, because the application id is part
of the local state path and changing it creates a fresh Streams application.
Use `spring.kafka.streams.properties.auto.offset.reset: earliest` only as the
cold-start fallback when no local checkpoint exists.

When the container runs as a non-root UID, avoid mounting the persistent volume
directly at the exact `state-dir` path. Kafka Streams sets POSIX permissions on
`state.dir` during startup, and a PVC mount root is often owned by root even when
`fsGroup` grants group write access. Prefer mounting the volume at a parent path
and configuring `state-dir` as a child directory created by the application UID,
or prepare the mounted directory with an init container that owns/chmods it.

## Reusable Insight
For `GlobalKTable`, Kafka Streams stores replicated table state under
`state.dir/<application-id>/global` and writes checkpoint offsets for recovery.
On restart, a surviving checkpoint lets the global consumer seek from the stored
offsets instead of rebuilding from the beginning. If the directory or checkpoint
is missing, the global table must bootstrap from the source topic again.

## Detection
Look for profile or base YAML that lacks `spring.kafka.streams.state-dir`, uses
the default `/tmp/kafka-streams`, or enables cleanup on startup/shutdown. In
Kubernetes, inspect whether the configured path is backed by an `emptyDir` or
image filesystem rather than a persistent volume. Also compare the Deployment
`volumeMounts[].mountPath` with `spring.kafka.streams.state-dir`: if they are
the same path and the pod runs as non-root, Kafka Streams can fail with
`StateDirectory` permission errors while changing directory permissions.

## Verification
Add configuration tests that bind `spring.kafka` and assert:

- `KafkaProperties.streams.stateDir` equals the configured path.
- `KafkaProperties.streams.cleanup.onStartup` is `false`.
- `KafkaProperties.streams.cleanup.onShutdown` is `false`.
- `auto.offset.reset` remains configured for cold starts if required.

For runtime verification, start the app, let a `GlobalKTable` finish
bootstrapping, shut down gracefully, and confirm the state directory contains
the application id plus global store checkpoint files before restarting.
