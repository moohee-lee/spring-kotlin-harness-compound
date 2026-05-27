---
title: Log Kafka Streams Deserialization Failure Payloads
tags: [springboot-kotlin, kafka-streams, logging, deserialization]
scope: cross-project
status: active
---

# Log Kafka Streams Deserialization Failure Payloads

## Context
Spring Boot Kotlin services using Kafka Streams can fail before business logic
sees a record when JSON deserialization fails. The built-in
`LogAndFailExceptionHandler` logs topic, partition, and offset, but not the raw
message payload. `GlobalKTable` joins add another trap: a deserialization error
can surface while reading the materialized state store, so a source-level
exception handler alone may not show the bad value.

## Wrong Direction
Do not rely only on the default Kafka Streams deserialization handler when an
incident requires the actual message body. Also do not add business-field logs
after the DSL join and assume they cover deserialization failures, because the
record may fail before it reaches those operators.

## Correct Pattern
Use two diagnostic layers:

1. Configure a custom `DeserializationExceptionHandler` that logs
   topic/partition/offset plus raw key and raw value previews, then returns
   `FAIL` if the existing behavior should remain fail-fast.
2. Wrap JSON `Serde` instances that back state stores or reference tables with
   a deserializer decorator. On `SerializationException`, log the configured
   source topic, the deserializer topic/store name, and the raw UTF-8 payload
   before rethrowing.

Keep payload previews bounded, for example 4 KB, and escape newlines so logs
remain single-event diagnostics.

## Reusable Insight
There are two different failure surfaces: source-record deserialization and
state-store value deserialization. Kafka metadata is strongest in the first
path; raw value logging at the serde boundary is what still works in the second
path.

## Detection
Look for stack traces with `JacksonJsonDeserializer`, `ValueAndTimestampDeserializer`,
`StateSerdes.valueFrom`, or `KTableSourceValueGetter` where logs show byte
arrays but not the JSON payload. If only `LogAndFailExceptionHandler` is
configured, the bad message body will likely be missing from application logs.

## Verification
Add unit tests that call the custom handler with a `ConsumerRecord<byte[], byte[]>`
and assert the logged key/value previews. Add a second test that invokes the
wrapped serde directly with malformed JSON and asserts the payload is logged even
when the topic argument is the materialized store name. Add profile property
tests to ensure Kubernetes/runtime profiles use the custom handler.
