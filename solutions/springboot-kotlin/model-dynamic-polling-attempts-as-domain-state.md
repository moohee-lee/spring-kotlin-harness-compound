---
title: 동적 polling attempt는 scheduler retry가 아니라 domain state로 모델링한다
tags: [springboot-kotlin, scheduler, polling, distributed-worker]
scope: cross-project
status: active
principles: [persistence-transactions]
---

# 동적 polling attempt는 scheduler retry가 아니라 domain state로 모델링한다

> 관련 공통 원칙: [영속성, 트랜잭션, 동시성 경계](../../principles/ko/persistence-transactions.md)

## 적용 시점

Spring Boot Kotlin service가 외부 caller로부터 동적 polling job을 받고, 각 job이 target URI, callback URI, interval, max attempts, expected-response predicate를 가질 때 적용한다. service가 여러 pod에서 실행되면서 중복 실행을 막고 work를 분산해야 하는 경우도 포함한다.

## 피해야 할 방향

사용자가 제출한 polling job마다 scheduler-level recurring job을 만들거나, scheduler exception retry를 business polling attempt counter로 쓰면 안 된다. expected response mismatch는 정상 business state이지 infrastructure failure가 아니다. recurring scheduler primitive는 limit, startup registration, missed-run semantics, cleanup behavior가 user-created polling lifecycle과 맞지 않을 수 있다.

## 권장 패턴

polling lifecycle을 domain table에 저장한다. status, attempt count, next poll time, lock owner/token, lease expiry, last response, callback status, final result를 domain state로 둔다. scheduler나 worker는 한 번에 한 attempt만 trigger한다. 각 attempt 후 domain state를 update하고 job을 finalize하거나 `now + interval`로 다음 attempt를 schedule/mark한다.

job library를 사용한다면 stable domain job id를 들고 있는 one-time scheduled execution을 선호한다. library retry policy는 unexpected infrastructure exception에만 사용하고, business retry는 polling job aggregate 안에 둔다. custom worker라면 row-level locking 또는 atomic update/returning query와 lease token으로 due row를 claim하고, HTTP call은 DB transaction 밖에서 수행한 뒤 같은 token으로 guarded update를 수행한다.

callback delivery가 polling result의 one-to-one continuation이고 retention, ownership, query requirement를 공유한다면 polling aggregate 안에 둘 수 있다. callback delivery가 independent lifecycle, multiple destination, separate retention/dead-letter, large payload/history, isolated worker/query scaling을 필요로 하면 `callback_delivery`나 outbox-style table로 분리한다.

callback이 HTTP callback, Kafka event publication, email, SMS 같은 여러 post-processing action type으로 확장되면 polling job column으로 늘리지 말고 child delivery/outbox row로 모델링한다. polling job은 target polling lifecycle과 final result를 소유하고, 각 delivery row는 type, destination/config, payload snapshot, status, attempts, lease, retry schedule, last error를 소유한다.

## 공통 원칙

동적 polling의 source of truth는 background scheduler의 retry/recurrence metadata가 아니라 domain job record다. 이렇게 해야 scheduler semantics와 business semantics가 섞이지 않고 callback decision과 multi-pod recovery를 감사 가능하게 만들 수 있다.

## 점검 방법

target response가 expected predicate와 맞지 않을 때 exception을 던지는 code, user별 polling interval을 recurring scheduler job으로 설정하는 code, scheduler retry count를 attempt count로 쓰는 code를 찾는다. DB claim transaction을 잡은 채 external HTTP call을 수행하는지도 확인한다.

## 검증 방법

두 worker가 due job을 동시에 claim해도 각 job이 최대 한 번만 처리되는지 test한다. unmatched target response가 scheduler failure가 아니라 domain attempt count 증가와 next attempt schedule로 이어지는지 확인한다. worker crash나 expired lease 후 다른 worker가 resume할 수 있고, callback delivery가 persisted callback state에서 idempotent하게 실행되거나 retry되는지도 검증한다.
