---
title: Avoid build-time env merging in Dockerfiles
tags: [springboot-kotlin, docker, jvm-options]
scope: cross-project
status: draft
---

# Avoid build-time env merging in Dockerfiles

## Context
Spring Boot JVM services often accept runtime tuning through environment
variables such as `JAVA_OPTS`, `JAVA_OPTIONS`, and `JVM_OPTIONS` in a
Dockerfile entrypoint.

## Wrong Direction
Do not try to merge runtime environment variables with a Dockerfile `ENV`
instruction such as `ENV JAVA_OPTIONS="${JAVA_OPTS} ${JAVA_OPTIONS}"`.
Docker evaluates that expression at build time, so variables supplied later
with `docker run -e` or Kubernetes env settings are not merged there. Docker's
static checker also reports `UndefinedVar` for variables that are not already
defined in the Dockerfile.

## Correct Pattern
Set only stable image defaults with `ENV`, then expand runtime variables in
the shell-form entrypoint:

```Dockerfile
ENV JAVA_OPTIONS=""
ENV JVM_OPTIONS="-XshowSettings:vm -XX:MaxRAMPercentage=60.0 -XX:+UseG1GC"

ENTRYPOINT ["sh","-c","exec java $JAVA_OPTS $JAVA_OPTIONS $JVM_OPTIONS -jar app.jar"]
```

This lets container runtime configuration override `JVM_OPTIONS` and supply
`JAVA_OPTS` or `JAVA_OPTIONS` without relying on build-time interpolation.

## Reusable Insight
Dockerfile `ENV` is image build metadata, not a runtime expression engine.
Runtime JVM flags should be combined where the runtime shell expands them.

## Detection
Run Docker's static Dockerfile check and look for `UndefinedVar` warnings on
`ENV` instructions that reference `JAVA_OPTS`, `JAVA_OPTIONS`, or similar
runtime-only variables.

## Verification
Use:

```bash
docker buildx build --check .
```

For local Apple Silicon machines using amd64-only corporate base images, an
unrelated `InvalidBaseImagePlatform` warning may remain unless the builder is
configured for the deployment platform. Confirm no `UndefinedVar` warning
remains for the JVM option handling.
