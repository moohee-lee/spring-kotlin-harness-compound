---
title: model-dynamic-polling-attempts-as-domain-state
tags: [springboot-kotlin, scheduler, polling, distributed-worker]
scope: cross-project
status: draft
---

# model-dynamic-polling-attempts-as-domain-state

## Context
Use this when a Spring Boot Kotlin service accepts dynamic polling jobs from external callers. Each job can have its own target URI, callback URI, interval, max attempts, and expected-response predicate. The service may run in multiple pods and must avoid duplicate execution while still distributing work.

## Wrong Direction
Do not model every user-submitted polling job as a scheduler-level recurring job or rely on scheduler exception retries as the business polling attempt counter. Expected response mismatch is a normal business state, not an infrastructure failure. Recurring scheduler primitives often have limits, startup registration assumptions, missed-run semantics, or cleanup behavior that do not map cleanly to user-created polling lifecycles.

## Correct Pattern
Store the polling lifecycle in domain tables: status, attempt count, next poll time, lock owner/token, lease expiry, last response, callback status, and final result. Let the scheduler or worker trigger one attempt at a time. After each attempt, update domain state and either finalize the job or schedule/mark the next attempt at `now + interval`.

If a job library is used, prefer one-time scheduled executions that carry a stable domain job id. Use the library retry policy only for unexpected infrastructure exceptions, and keep business retries in the polling job aggregate. For a custom worker, claim due rows with row-level locking or an atomic update/returning query plus a lease token, perform HTTP outside the DB transaction, then complete the attempt with a guarded update using the same token.

Keep callback state on the polling aggregate while callback delivery is a one-to-one continuation of the polling result and shares the same retention, ownership, and query needs. Split it into a `callback_delivery` or outbox-style table when callback delivery needs an independent lifecycle, multiple destinations, separate retention/dead-letter handling, large payload/history storage, or isolated worker/query scaling.

When "callback" evolves into multiple post-processing action types such as HTTP callback, Kafka event publication, email, SMS, or future delivery channels, model those as child delivery/outbox rows rather than columns on the polling job. The polling job owns the target polling lifecycle and final result; each post-processing row owns one delivery action's type, destination/config, payload snapshot, status, attempts, lease, retry schedule, and last error. This keeps polling completion atomic with enqueueing follow-up work while allowing each delivery channel to scale, retry, and fail independently.

For library selection, prefer a small embedded DB scheduler or a custom DB-lease worker when the domain table is already the source of truth. Avoid modeling each user-submitted polling lifecycle as a scheduler-level recurring job unless the library can cleanly represent per-job stop conditions, cleanup, startup registration, missed-run behavior, and transaction boundaries. Use richer background-job libraries mainly when their dashboard, operational controls, or general job-processing features are worth the extra scheduler metadata.

## Reusable Insight
The source of truth for dynamic polling should be the domain job record, not the background scheduler's retry or recurrence metadata. This avoids dual semantics, gives reliable callback decisions, and makes multi-pod recovery auditable.

## Detection
Look for code that throws an exception when the target response does not match the expected predicate, configures per-user polling intervals as recurring scheduler jobs, or increments attempt count based on scheduler retry count. Also look for external HTTP calls performed while holding the DB claim transaction.

## Verification
Test that two workers claiming due jobs concurrently process each job at most once. Test that an unmatched target response increments the domain attempt count and schedules the next attempt without marking scheduler failure. Test that a worker crash or expired lease allows another worker to resume, and that callback delivery is idempotent or retried from persisted callback state.
