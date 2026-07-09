# 설정 소유권과 로딩 시점

## 원칙

설정은 값을 사용하는 코드보다 먼저, 그리고 그 값을 소유하는 실행 환경에서 로딩되어야 한다. base 설정은 모든 환경에서 참인 값만 담고, profile, ConfigMap, Vault, 환경 변수는 각자 맡은 책임만 가져야 한다.

## 적용 기준

- datasource, Kafka, Vault, 외부 endpoint처럼 환경마다 달라지는 값은 profile이나 배포 설정이 소유한다.
- secret은 Vault나 secret backend가 소유하고, non-secret runtime endpoint는 ConfigMap이나 profile 설정이 소유한다.
- `spring.config.import`는 import 대상 안에 넣지 않는다. import 자체가 먼저 활성화되어야 한다.
- 여러 외부 설정 source를 합칠 때는 충돌 가능한 key에 접두사를 붙인다.
- 테스트 profile은 실제 Vault, Kubernetes API, 내부 endpoint를 우연히 호출하지 않게 import와 URL을 차단한다.

## 피해야 할 신호

- base `application.yaml`에 `${ENV:default}` 형태의 환경별 endpoint가 많다.
- 테스트가 통과하지만 로그에 Vault login, secret read, Kubernetes ConfigMap read가 보인다.
- `cluster.local`, Vault path, Kubernetes namespace, profile 이름을 서로 같은 의미로 가정한다.
- Helm value file을 Kubernetes manifest처럼 생각한다.

## 검증 방법

- base/profile YAML을 분리해서 로딩하는 configuration test를 둔다.
- Spring Boot `Binder`로 실제 runtime property shape까지 검증한다.
- Helm render, ConfigMap manifest, Pod namespace, RBAC, service account를 함께 확인한다.
- 테스트 profile에서는 실제 외부 endpoint 대신 inert local URL이나 test-owned stub URL만 허용한다.
