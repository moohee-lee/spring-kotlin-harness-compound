---
title: Align detekt version when overriding Kotlin in skeleton overlays
tags: [springboot-kotlin, harness, detekt, kotlin, initializr]
scope: cross-project
status: active
---

# Align detekt version when overriding Kotlin in skeleton overlays

## Context

Use this when a Spring Boot Kotlin Initializr scaffold preserves a user-requested Kotlin version while applying a reusable skeleton overlay that also brings detekt configuration.

## Wrong Direction

Only copying the skeleton `PluginVersions.DETEKT` value or only preserving `PluginVersions.KOTLIN` can create a compiler mismatch. For example, detekt `2.0.0-alpha.2` is compiled with Kotlin `2.3.0`; running it in a project whose Kotlin plugin was intentionally overridden to `2.4.0` fails before static analysis can report code issues.

## Correct Pattern

Treat detekt as a tool-specific compatibility dependency. Keep the user-requested Kotlin version unchanged, check detekt's official compatibility table, then update `PluginVersions.DETEKT` and any `detektPlugins(...)` artifacts, such as `dev.detekt:detekt-rules-ktlint-wrapper`, to the matching detekt version.

For Kotlin `2.4.0`, detekt's compatibility table lists detekt `2.0.0-alpha.5` with Gradle `9.5.1` and JDK `25`, so the overlay should use that detekt version instead of lowering Kotlin.

## Reusable Insight

Skeleton overlay version preservation is not enough by itself. Build tools that embed or run a Kotlin compiler need their own compatibility alignment when the target project overrides the skeleton's Kotlin version.

## Detection

Run `./gradlew detekt --stacktrace` after overlay. The mismatch usually appears as `detekt was compiled with Kotlin <x> but is currently running with <y>`. Also compare `buildSrc/src/main/kotlin/PluginVersions.kt` against the official detekt compatibility table whenever `PluginVersions.KOTLIN` differs from the skeleton source.

## Verification

Run:

```bash
env GRADLE_USER_HOME=.gradle-user-home ./gradlew detekt --no-daemon --stacktrace
env GRADLE_USER_HOME=.gradle-user-home ./gradlew build --no-daemon --stacktrace
```

Both commands should pass without the detekt Kotlin compiler mismatch. If detekt then reports ordinary style issues, fix those source/config issues separately from the tool-version alignment.
