# Kafka Streams 운영 계약

## 원칙

Kafka Streams는 topology 코드만으로 운영되지 않는다. state directory, application id, serde failure surface, native runtime, shutdown budget이 함께 맞아야 안정적으로 재시작하고 장애를 진단할 수 있다.

## 적용 기준

- `GlobalKTable`이나 state store를 쓰면 `state.dir`를 안정적인 persistent path로 둔다.
- `application-id` 변경은 rename이 아니라 새 Streams application 생성으로 취급한다.
- source deserialization failure와 state-store value deserialization failure를 구분해서 logging한다.
- incident log에는 topic, partition, offset, bounded payload preview를 남긴다.
- RocksDB-backed store를 쓰는 image에는 C++ runtime을 포함한다.
- Kubernetes grace period는 Kafka Streams close timeout보다 충분히 길게 잡는다.

## 피해야 할 신호

- `state.dir`가 `/tmp`이거나 pod ephemeral filesystem에 있다.
- `cleanup.on-startup`이나 `cleanup.on-shutdown`이 state 복구 의도와 충돌한다.
- deserialization 장애 로그에 payload나 record 위치가 없다.
- `GlobalKTable` topic을 metadata logging 때문에 `KStream`으로 중복 등록한다.
- `application-id` 변경과 `auto.offset.reset=earliest`가 함께 들어가는데 replay 영향 분석이 없다.

## 검증 방법

- Spring Kafka properties binding test로 `state-dir`, cleanup, dotted client property를 확인한다.
- `TopologyTestDriver`와 log appender로 metadata logging을 검증한다.
- image smoke test로 RocksDB native dependency를 확인한다.
- rollout 또는 pod deletion 시 graceful shutdown log와 exit code를 확인한다.
