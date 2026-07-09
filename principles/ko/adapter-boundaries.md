# 어댑터 경계와 재사용 구조

## 원칙

외부 시스템과 만나는 코드는 adapter boundary에서 모으고, domain/application layer에는 transport, framework, client-specific 세부 사항을 흘려보내지 않는다. 재사용 가능한 adapter infrastructure는 처음부터 특정 domain 이름에 갇히지 않게 둔다.

## 적용 기준

- Feign, WebClient, signing interceptor 같은 client infrastructure는 공통 adapter base 아래에 둔다.
- request 값의 primitive validation은 web boundary에서 끝내고, application service는 use-case policy와 domain rule을 다룬다.
- utility-only artifact와 Spring Boot starter/autoconfigure artifact는 분리한다.
- UUID 생성, signing helper처럼 단순하고 공통적인 함수는 작은 wrapper 뒤에 숨긴다.
- 관측성은 기본적으로 runtime instrumentation을 우선하고, manual span은 필요한 곳에만 추가한다.

## 피해야 할 신호

- `@EnableFeignClients`가 특정 gateway package만 scan한다.
- interceptor가 global `@Component`로 등록되어 모든 client에 적용된다.
- application service가 request DTO primitive constraint를 `require`로 다시 검사한다.
- utility 하나를 쓰려고 starter jar 전체를 runtime classpath에 올린다.
- 실질 정책 없이 property를 순수 함수에 넘기기만 하는 Spring service가 생긴다.

## 검증 방법

- adapter scan 범위, opt-in client configuration, interceptor 적용 범위를 focused test로 확인한다.
- handler/router test로 invalid request가 use case 호출 전에 400으로 변환되는지 검증한다.
- shared dependency jar에 Boot auto-configuration metadata나 logging resource가 없는지 확인한다.
- 생성 ID 기본값이나 wrapper 함수가 의도한 UUID version을 쓰는지 테스트한다.
