---
title: Limit Docker build context for jar only Spring Boot images
tags: [springboot-kotlin, docker, github-actions]
scope: cross-project
status: draft
---

# Limit Docker build context for jar only Spring Boot images

## Context
Spring Boot services that build a jar with Gradle and use a runtime-only
Dockerfile often need only `Dockerfile` and `build/libs/*.jar` in the Docker
build context.

## Wrong Direction
Do not leave the repository without a `.dockerignore` when the Dockerfile only
copies the built jar. A GitHub Actions Docker build that runs after Gradle can
otherwise upload `.git`, `.gradle`, generated sources, test reports, compiled
classes, and other build output into the BuildKit context even though the image
does not need them.

## Correct Pattern
Add a deny-by-default `.dockerignore` and allow only the runtime image inputs:

```dockerignore
**
!Dockerfile
!build/
!build/libs/
!build/libs/*.jar
```

Run the Gradle jar task before Docker build, for example `./gradlew clean
bootJar`, so the allowlisted jar exists before BuildKit loads the context.

## Reusable Insight
For jar-only Spring Boot Dockerfiles, build context size should be treated as a
CI contract. The Dockerfile input set is small and explicit, so the ignore file
should make that boundary visible.

## Detection
Run a local `docker buildx build ... .` and inspect the `load build context`
line. If the context is tens or hundreds of MB for a Dockerfile that only does
`COPY ./build/libs/*.jar`, the repository is likely missing a focused
`.dockerignore`.

## Verification
Run:

```bash
./gradlew clean bootJar
docker buildx build --platform linux/amd64 -t <service>:workflow-check --load .
```

The Docker build should still copy `build/libs/*.jar`, while the BuildKit log
should show a tiny transferred context rather than the full repository and
Gradle build tree.
