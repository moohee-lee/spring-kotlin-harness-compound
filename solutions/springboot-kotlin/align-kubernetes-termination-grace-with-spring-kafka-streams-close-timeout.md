---
title: Align Kubernetes termination grace with Spring Kafka Streams close timeout
tags: [springboot-kotlin, kubernetes, kafka-streams, graceful-shutdown]
scope: cross-project
status: draft
---

# Align Kubernetes termination grace with Spring Kafka Streams close timeout

## Context
Spring Boot Kotlin services running Kafka Streams on Kubernetes may use
Deployment `strategy.type: Recreate` and persistent Kafka Streams state
directories. Kafka Streams shutdown is coordinated through Spring lifecycle
management and `StreamsBuilderFactoryBean.closeTimeout`, while Kubernetes
controls how long the container may run after termination starts.

## Wrong Direction
Do not leave Kubernetes `terminationGracePeriodSeconds` at the default 30
seconds when application code configures Kafka Streams close timeout to 60
seconds or longer. Also do not add a `preStop` sleep without increasing the
termination grace period; Kubernetes starts the grace countdown before the
`preStop` hook runs, so hook time consumes the same shutdown budget.

## Correct Pattern
Set the Kubernetes grace period greater than the sum of Spring shutdown phase
timeout, Kafka Streams close timeout, and any `preStop` drain delay. Enable
Spring Boot graceful shutdown for inbound HTTP and align
`spring.lifecycle.timeout-per-shutdown-phase` with the Kafka Streams close
timeout. Keep Kafka Streams `cleanup.on-shutdown=false` when state is persisted.
Use `preStop` only for endpoint or load-balancer drain delay, not as the main
Kafka Streams shutdown mechanism.

## Reusable Insight
Graceful shutdown is a budget alignment problem across Kubernetes, Spring
Boot, and Kafka Streams. The smallest timeout wins; if Kubernetes kills the
container before Spring/Kafka Streams finish closing, offsets and local state
may not settle cleanly.

## Detection
Look for `strategy.type: Recreate`, `terminationGracePeriodSeconds`, Spring
Boot `server.shutdown`, `spring.lifecycle.timeout-per-shutdown-phase`,
`StreamsBuilderFactoryBean.setCloseTimeout`, Kafka Streams `state-dir`, and
`cleanup.on-shutdown`. Flag any configuration where Kubernetes grace is less
than or equal to the application close timeout.

## Verification
Render Helm and confirm the Pod spec contains the intended
`terminationGracePeriodSeconds` and optional lifecycle hook. During rollout or
manual pod deletion, verify the process receives SIGTERM, readiness stops
accepting traffic, Kafka Streams logs a normal shutdown, no SIGKILL/exit 137 is
observed, and the new pod restores from the persisted state directory without a
full rebuild.
