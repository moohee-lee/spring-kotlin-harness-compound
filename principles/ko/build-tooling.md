# 빌드와 도구 체인은 런타임을 오염시키지 않는다

## 원칙

빌드 도구, codegen, static analysis, local developer runtime은 애플리케이션 실행 경로와 분리되어야 한다. 편의를 위해 compile이나 bootRun에 붙인 도구 의존성이 Docker, Testcontainers, 특정 JDK/Kotlin compiler 요구사항을 일상 실행 경로로 끌고 오면 안 된다.

## 적용 기준

- Testcontainers-backed jOOQ codegen은 명시적 schema generation workflow로 둔다.
- `bootRun`이나 일반 compile이 Docker/Testcontainers를 필요로 하는지 dry-run으로 확인한다.
- Kotlin version을 overlay로 보존하면 detekt 등 compiler-embedded tool version도 함께 맞춘다.
- Spring Boot virtual thread 같은 framework-level 설정은 공식 property로 켠다.
- Postman 같은 외부 도구 script도 현재 sandbox가 지원하는 API를 기준으로 관리한다.

## 피해야 할 신호

- `compileKotlin`이 무조건 `jooqCodegen`에 의존한다.
- `./gradlew bootRun --dry-run`에 Docker/Testcontainers task가 끼어 있다.
- Kotlin만 upgrade하고 detekt나 detekt plugin version은 skeleton 값을 그대로 둔다.
- 오래된 Postman `crypto-js` 예제를 그대로 사용한다.

## 검증 방법

- `./gradlew bootRun --dry-run`으로 runtime startup path를 확인한다.
- `./gradlew detekt --stacktrace`로 tool/compiler compatibility를 확인한다.
- framework property는 configuration binding test나 context load test로 검증한다.
- 외부 도구 script는 backend/reference signer와 같은 golden vector로 비교한다.
