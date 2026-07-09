---
title: Configure Rancher Desktop for Testcontainers
tags: [springboot-kotlin, testcontainers, rancher-desktop, jooq, gradle]
scope: cross-project
status: draft
---

# Configure Rancher Desktop for Testcontainers

## Context

Spring Boot Kotlin projects may use Testcontainers from Gradle tasks, tests, or
jOOQ code generation. When the developer runtime is Rancher Desktop on macOS,
the Docker CLI can work through the `rancher-desktop` Docker context while
Testcontainers still fails to discover a usable Docker environment.

## Wrong Direction

Do not assume `docker info` success means Testcontainers will inherit the Docker
CLI context. Testcontainers discovers Docker through environment variables and
well-known socket paths. On Rancher Desktop, the host socket is often
`$HOME/.rd/docker.sock`, not `/var/run/docker.sock`.

## Correct Pattern

For Rancher Desktop with the moby/dockerd engine, export Testcontainers-specific
environment before running Gradle:

```bash
export DOCKER_HOST="unix://$HOME/.rd/docker.sock"
export TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE=/var/run/docker.sock
export TESTCONTAINERS_HOST_OVERRIDE=$(rdctl shell ip a show vznat | awk '/inet / {sub("/.*",""); print $2}')
```

If Rancher Desktop provides `rdctl info --field ip-address` in the installed
version, that command can be used for `TESTCONTAINERS_HOST_OVERRIDE` instead.
Persist the exports in the developer shell profile when the project expects
frequent Testcontainers-backed Gradle tasks.

## Reusable Insight

Docker CLI context and Testcontainers Docker discovery are separate paths. A
project can fail with "Could not find a valid Docker environment" even while
`docker context ls` and `docker info` are healthy.

## Detection

Check all three boundaries:

```bash
docker context ls
docker info
printenv DOCKER_HOST
ls -l /var/run/docker.sock "$HOME/.rd/docker.sock"
```

If `docker info` works through `rancher-desktop`, `DOCKER_HOST` is empty, and
`/var/run/docker.sock` is missing, Testcontainers will usually miss the daemon.
If setting only `DOCKER_HOST` changes the failure to a Ryuk mount error for
`$HOME/.rd/docker.sock`, add `TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE`.

## Verification

Run the original Testcontainers-backed command with the environment variables:

```bash
DOCKER_HOST="unix://$HOME/.rd/docker.sock" \
TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE=/var/run/docker.sock \
TESTCONTAINERS_HOST_OVERRIDE="$(rdctl shell ip a show vznat | awk '/inet / {sub("/.*",""); print $2}')" \
./gradlew jooqCodegen
```

Success proves the issue was environment discovery rather than the jOOQ
generator configuration itself.
