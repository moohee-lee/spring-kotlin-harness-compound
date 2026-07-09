# 테스트는 계약을 검증해야 한다

## 원칙

테스트는 같은 구현을 다시 호출해서 자기 자신을 검증하면 안 된다. 외부 protocol, profile binding, Spring context startup, integration boundary는 각각 독립적인 oracle이나 실제 binding 경로로 확인해야 한다.

## 적용 기준

- porting한 signing, hashing, canonicalization logic은 reference implementation이나 protocol 문서에서 만든 golden vector로 검증한다.
- Spring context test는 eager singleton dependency를 실제 seed data나 test fake로 명시적으로 만족시킨다.
- test profile은 실제 외부 서비스 URL을 사용하지 않는다.
- integration test는 routing, SQL type, schema bootstrap 같은 작은 실패 지점을 분리해서 잡는다.
- Docker/Testcontainers가 필요한 테스트는 개발자 runtime discovery까지 별도 계약으로 다룬다.

## 피해야 할 신호

- expected value를 production helper가 다시 계산한다.
- `@SpringBootTest`가 disabled runner만 믿고 constructor-time DB dependency를 준비하지 않는다.
- `application-test.yaml`에 dev, stage, internal host가 들어간다.
- H2 named database에서 같은 schema script가 여러 context에 반복 적용되는데 idempotent하지 않다.

## 검증 방법

- golden vector에는 canonical input, intermediate string, final signature/header를 함께 둔다.
- context startup test와 focused unit test를 모두 둔다.
- test profile configuration test로 실제 endpoint가 들어오지 않음을 확인한다.
- Testcontainers 환경은 Docker CLI context와 `DOCKER_HOST`/socket discovery를 따로 확인한다.
