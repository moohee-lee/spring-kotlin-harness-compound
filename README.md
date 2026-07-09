# Spring Boot Kotlin Harness Compound

이 저장소는 Spring Boot Kotlin 서비스에서 반복해서 적용할 수 있는 운영, 설정, 테스트, 어댑터, 빌드 원칙을 정리한 지식 베이스입니다.

기존 문서들은 특정 장애나 구현 사례에서 출발했지만, 이 저장소에서는 다음 기준으로 재정리합니다.

- 특정 프로젝트 이름이나 한 번의 사고에만 묶이지 않는 공통 규칙으로 쓴다.
- 적용 시점, 피해야 할 방향, 권장 패턴, 점검 방법, 검증 방법을 같은 구조로 둔다.
- 세부 레시피는 공통 원칙을 참조하고, 공통 원칙은 여러 레시피를 묶는 기준이 된다.
- 설정, 런타임, 테스트, 영속성, 어댑터, 빌드 경계를 섞지 않는다.

## 공통 원칙

- [설정 소유권과 로딩 시점](principles/ko/configuration-ownership.md)
- [런타임 경계와 컨테이너 계약](principles/ko/runtime-boundaries.md)
- [영속성, 트랜잭션, 동시성 경계](principles/ko/persistence-transactions.md)
- [Kafka Streams 운영 계약](principles/ko/kafka-streams-operations.md)
- [테스트는 계약을 검증해야 한다](principles/ko/testing-contracts.md)
- [어댑터 경계와 재사용 구조](principles/ko/adapter-boundaries.md)
- [빌드와 도구 체인은 런타임을 오염시키지 않는다](principles/ko/build-tooling.md)

## 레시피 분류

### 설정과 배포

- [프로필 소유 인프라 설정은 base YAML에서 분리한다](solutions/springboot-kotlin/keep-profile-owned-infrastructure-config-out-of-base-yaml.md)
- [Vault KV 비밀 값은 접두사로 네임스페이스를 만든다](solutions/springboot-kotlin/prefix-vault-kv-secrets-with-generic-keys.md)
- [테스트에서는 Vault ConfigData import를 프로필로 차단한다](solutions/springboot-kotlin/profile-gate-vault-configdata-imports-for-tests.md)
- [Vault Kubernetes 인증은 프로필별 토큰 파일 fallback을 둔다](solutions/springboot-kotlin/vault-kubernetes-auth-profile-token-file-fallback.md)
- [Spring Cloud Kubernetes ConfigData에는 스타터, RBAC, 비순환 import가 필요하다](solutions/springboot-kotlin/spring-cloud-kubernetes-configdata-needs-starter-rbac-and-non-circular-imports.md)
- [Helm value file은 단일 ConfigMap 템플릿을 여러 리소스로 만들지 못한다](solutions/springboot-kotlin/helm-value-files-cannot-create-multiple-single-configmap-resources.md)
- [cluster.local 사용 전 Kubernetes 서비스 DNS 도메인을 확인한다](solutions/springboot-kotlin/verify-kubernetes-service-dns-domain-before-using-cluster-local.md)
- [Spring Boot Kafka dotted client property는 Binder로 검증한다](solutions/springboot-kotlin/verify-spring-boot-kafka-dotted-client-properties-with-binder.md)

### Kafka Streams와 운영 런타임

- [Kafka Streams GlobalKTable state.dir를 영속화한다](solutions/springboot-kotlin/persist-kafka-streams-globalktable-state-dir.md)
- [Kafka Streams 역직렬화 실패 payload를 남긴다](solutions/springboot-kotlin/log-kafka-streams-deserialization-failure-payloads.md)
- [Kafka Streams 진단 로그에는 record metadata를 포함한다](solutions/springboot-kotlin/kafka-streams-diagnostic-record-metadata-logging.md)
- [Kubernetes 종료 grace와 Kafka Streams close timeout을 맞춘다](solutions/springboot-kotlin/align-kubernetes-termination-grace-with-spring-kafka-streams-close-timeout.md)
- [RocksDB JNI 이미지에는 C++ 런타임을 설치한다](solutions/springboot-kotlin/install-c-runtime-for-kafka-streams-rocksdb-jni-images.md)

### 영속성과 동시성

- [jOOQ persistence adapter는 codegen reference를 사용한다](solutions/springboot-kotlin/use-jooq-codegen-references-in-persistence-adapters.md)
- [jOOQ insert와 select mapping은 generated table record로 모은다](solutions/springboot-kotlin/map-jooq-insert-and-select-through-generated-table-records.md)
- [DB lease claim에는 가능하면 jOOQ DSL을 사용한다](solutions/springboot-kotlin/prefer-jooq-dsl-for-database-lease-claims.md)
- [독립 I/O만 병렬화하고 Spring JDBC DB 접근은 트랜잭션 단위로 유지한다](solutions/springboot-kotlin/parallelize-independent-io-without-parallelizing-spring-jdbc-db-access.md)
- [동적 polling attempt는 scheduler retry가 아니라 domain state로 모델링한다](solutions/springboot-kotlin/model-dynamic-polling-attempts-as-domain-state.md)
- [Coroutine worker의 in-flight 작업은 semaphore로 제한한다](solutions/springboot-kotlin/bound-coroutine-worker-in-flight-work-with-semaphores.md)

### 테스트 계약

- [Porting한 signing utility는 외부 golden vector로 검증한다](solutions/testing/use-external-golden-vectors-for-ported-signing-utilities.md)
- [Spring context test는 eager DB-backed bean의 의존성을 명시적으로 만족시킨다](solutions/springboot-kotlin/satisfy-eager-db-backed-beans-in-spring-context-tests.md)
- [Spring WebFlux jOOQ integration test를 작은 실패 지점으로 단단하게 만든다](solutions/springboot-kotlin/harden-spring-webflux-jooq-integration-tests.md)
- [외부 client test URL은 inert local URL을 사용한다](solutions/springboot-kotlin/use-inert-test-urls-for-external-clients.md)
- [Rancher Desktop 환경에서는 Testcontainers discovery 경로를 명시한다](solutions/springboot-kotlin/configure-rancher-desktop-for-testcontainers.md)

### 어댑터와 애플리케이션 구조

- [Feign client는 재사용 가능한 adapter base 아래에 둔다](solutions/springboot-kotlin/keep-feign-clients-under-reusable-adapter-base.md)
- [Request 값 검증은 web boundary에 둔다](solutions/springboot-kotlin/keep-request-value-validation-at-web-boundary.md)
- [Utility-only library와 Spring Boot starter는 분리한다](solutions/dependencies/split-util-only-libraries-from-spring-boot-starters.md)
- [UUID v7 생성은 java.util.UUID wrapper 함수 뒤에 숨긴다](solutions/springboot-kotlin/wrap-kotlin-uuid-v7-generation-behind-a-java-uuid-top-level-function.md)
- [Coroutine Spring Boot 서비스의 기본 관측성은 OpenTelemetry Java agent로 둔다](solutions/springboot-kotlin/opentelemetry-java-agent-is-the-default-for-coroutine-spring-boot-services.md)

### 빌드와 도구

- [Dockerfile에서 build-time ENV merge를 하지 않는다](solutions/springboot-kotlin/avoid-build-time-env-merging-in-dockerfiles.md)
- [jar-only Spring Boot image는 Docker build context를 제한한다](solutions/springboot-kotlin/limit-docker-build-context-for-jar-only-spring-boot-images.md)
- [bootRun compile path가 Testcontainers jOOQ codegen에 의존하지 않게 한다](solutions/springboot-kotlin/do-not-make-bootrun-compile-depend-on-testcontainers-jooq-codegen.md)
- [Skeleton overlay에서 Kotlin을 override하면 detekt 버전도 맞춘다](solutions/springboot-kotlin/align-detekt-version-when-overriding-kotlin-in-skeleton-overlays.md)
- [Spring Boot virtual thread는 application YAML로 활성화한다](solutions/springboot-kotlin/enable-spring-boot-virtual-threads-with-application-yaml.md)
- [Postman signing script는 Web Crypto를 사용한다](solutions/tooling/use-web-crypto-for-postman-signing-scripts.md)
