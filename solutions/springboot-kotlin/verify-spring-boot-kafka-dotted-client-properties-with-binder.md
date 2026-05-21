---
title: Verify Spring Boot Kafka dotted client properties with Binder
tags: [springboot-kotlin, kafka, configuration, binder]
scope: cross-project
status: active
---

# Verify Spring Boot Kafka dotted client properties with Binder

## Context

Spring Boot Kafka configuration often stores raw Kafka client properties under
`spring.kafka.properties`, `spring.kafka.producer.properties`, and
`spring.kafka.streams.properties`. Kafka Streams also supports dotted client
prefixes such as `main.consumer.sasl.jaas.config`,
`global.consumer.sasl.jaas.config`, `producer.sasl.jaas.config`, and
`admin.sasl.jaas.config`.

## Wrong Direction

Only checking the YAML text or a flattened `YamlPropertySourceLoader` key can
miss binder-level issues. A dotted key may appear in the file but still fail to
bind to the `KafkaProperties` map in the shape Spring Boot uses at runtime.

## Correct Pattern

For Spring Boot 4, bind the loaded environment to
`org.springframework.boot.kafka.autoconfigure.KafkaProperties` and assert the
resulting maps contain the exact Kafka client keys. In Spring Boot 4 the package
is `org.springframework.boot.kafka.autoconfigure`, not the older
`org.springframework.boot.autoconfigure.kafka` package.

## Reusable Insight

Dotted Kafka client property keys are safest when tests verify both:

- the config file contains the intended keys and defaults
- `Binder.get(environment).bind("spring.kafka", KafkaProperties::class.java)`
  produces maps with keys such as `security.protocol`,
  `main.consumer.sasl.jaas.config`, and `producer.sasl.jaas.config`

## Detection

Use this pattern when adding SCRAM/SASL, per-client Kafka Streams credentials,
or any Kafka property whose key itself contains dots.

## Verification

Run a focused configuration test that loads `application.yaml`, adds the
property source to a `StandardEnvironment`, binds `spring.kafka` to
`KafkaProperties`, and asserts the common, producer, and streams property maps.
