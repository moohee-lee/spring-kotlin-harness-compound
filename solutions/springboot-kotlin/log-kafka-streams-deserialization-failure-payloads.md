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

When a topic is already registered as a `GlobalKTable`, do not register the same
topic again as a normal `KStream` just to log source offsets. Kafka Streams
rejects duplicate source-topic registration.

## Correct Pattern

Use diagnostic layers that match where parsing happens:

1. Configure a custom `DeserializationExceptionHandler` that logs
   topic/partition/offset plus raw key and raw value previews, then returns
   `FAIL` if the existing behavior should remain fail-fast.
2. Wrap JSON `Serde` instances that back state stores or reference tables with
   a deserializer decorator. On `SerializationException`, log the configured
   source topic, the deserializer topic/store name, and the raw UTF-8 payload
   before rethrowing.
3. If the Kafka contract is `String` key/value and JSON is parsed manually in
   DSL code, keep source and materialized values as `String` and call the
   project's JSON extension in the processing lambda. For `KStream` sources,
   capture `recordMetadata()` in a processor before parsing. For `GlobalKTable`
   sources, remember that the joiner only sees the materialized value, not the
   source offset; validate and log malformed raw table updates at the source
   hook, such as a `TimestampExtractor`, or move to a custom global store
   processor when richer metadata handling is required.

Keep payload previews bounded, for example 4 KB, and escape newlines so logs
remain single-event diagnostics.

## Reusable Insight

There are two different failure surfaces: source-record deserialization and
state-store value deserialization. Kafka metadata is strongest in the first
path; raw value logging at the serde boundary is what still works in the second
path.

When using String serdes to avoid serde-time failure, the parser is now in
application code, so the skip path must return an empty result instead of
throwing from `mapValues`, `selectKey`, or a joiner.

## Detection

Look for stack traces with `JacksonJsonDeserializer`,
`ValueAndTimestampDeserializer`, `StateSerdes.valueFrom`, or
`KTableSourceValueGetter` where logs show byte arrays but not the JSON payload.
If only `LogAndFailExceptionHandler` is configured, the bad message body will
likely be missing from application logs.

For String serde topologies, look for DSL lambdas that call JSON parsing and
then throw on null parse results. Those exceptions are processing errors and can
stop Kafka Streams unless the lambda returns no output for the bad record.

## Verification

Add unit tests that call the custom handler with a `ConsumerRecord<byte[], byte[]>`
and assert the logged key/value previews. Add a second test that invokes the
wrapped serde directly with malformed JSON and asserts the payload is logged
even when the topic argument is the materialized store name. Add profile
property tests to ensure Kubernetes/runtime profiles use the custom handler.

For String serde topologies, use `TopologyTestDriver` with `StringSerializer`
inputs and assert that malformed source/reference values log topic, partition,
offset, and raw value while later valid records still produce output.
