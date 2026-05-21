---
title: Kafka Streams Diagnostic Record Metadata Logging
tags: [springboot-kotlin, kafka-streams, logging, observability]
scope: cross-project
status: active
---

# Kafka Streams Diagnostic Record Metadata Logging

## Context

Spring Boot Kotlin services using Kafka Streams often need incident logs that identify the source Kafka record behind a failed mapping, join, or publish decision. The DSL operators after `stream()` usually do not expose topic, partition, and offset directly.

## Wrong Direction

Logging only the transformed DTO or only the thrown exception makes it hard to replay or inspect the failing Kafka record. Adding ad hoc offset fields into business DTOs also leaks transport metadata into domain-shaped message models.

## Correct Pattern

Capture Kafka metadata immediately after reading the input topic with a small processor based on `FixedKeyProcessor` and `FixedKeyProcessorContext.recordMetadata()`. Wrap the value with an internal processing record that carries nullable metadata plus the original message summary. Downstream DSL steps can then include `topic`, `partition`, and `offset` in WARN/ERROR logs without changing the external message contract.

Keep high-volume per-record payload and flow logs at TRACE, especially raw record receipt, candidate extraction, join observations, and produced-record traces. Reserve DEBUG for lower-volume diagnostic state that a developer would reasonably keep enabled during normal development. Use WARN only for abnormal records that require attention, not for expected unmatched reference data. Use ERROR when a record cannot be converted and the exception will be rethrown.

## Reusable Insight

Treat Kafka location as diagnostic context, not business data. Attach it at the boundary, carry it through internal processing, and log compact summaries of the source payload and joined reference data only at a level appropriate for the record volume.

## Detection

Look for Kafka Streams topologies where filter/map/join failures log messages like "skipped" or "failed" without the source topic/partition/offset or enough source fields to identify the bad input.

## Verification

Use `TopologyTestDriver` plus a Logback `ListAppender` to assert that skipped or mismatched records log the expected Kafka metadata and compact source data. Add profile configuration tests so production keeps application logging at INFO while development profiles enable DEBUG for troubleshooting.
