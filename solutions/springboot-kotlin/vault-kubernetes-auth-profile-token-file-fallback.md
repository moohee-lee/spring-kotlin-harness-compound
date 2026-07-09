---
title: Vault Kubernetes 인증은 프로필별 토큰 파일 fallback을 둔다
tags: [springboot-kotlin, vault, kubernetes, configdata, profiles]
scope: cross-project
status: active
principles: [configuration-ownership]
---

# Vault Kubernetes 인증은 프로필별 토큰 파일 fallback을 둔다

> 관련 공통 원칙: [설정 소유권과 로딩 시점](../../principles/ko/configuration-ownership.md)

## 적용 시점

Spring Cloud Vault ConfigData와 Kubernetes authentication으로 Vault secret을 import하는 service가 local 실행 profile과 Kubernetes 실행 profile을 모두 가질 때 적용한다. local profile은 exported service account token file을 사용하고, Kubernetes profile은 pod-mounted default token path를 사용할 수 있다.

## 피해야 할 방향

`curl`이나 Vault CLI로 Vault login을 검증한 뒤 Spring도 모든 active profile에서 같은 token file을 사용할 것이라고 가정하면 안 된다. active profile file에 `spring.cloud.vault.kubernetes.service-account-token-file`이 없으면 Spring Cloud Vault는 `/var/run/secrets/kubernetes.io/serviceaccount/token`으로 fallback한다. 이 path는 Kubernetes pod 안에서만 존재한다.

## 권장 패턴

Vault import path와 Kubernetes authentication setting은 Vault를 활성화할 수 있는 각 profile에 둔다. local과 Kubernetes에서 모두 쓰는 profile이라면 explicit token file fallback을 둔다.

```yaml
spring:
  cloud:
    vault:
      authentication: kubernetes
      kubernetes:
        role: <vault-role>
        kubernetes-path: <auth-mount>
        service-account-token-file: ${VAULT_K8S_TOKEN_FILE:/var/run/secrets/kubernetes.io/serviceaccount/token}
```

어떤 profile이 Kubernetes-only라면 local run은 local profile을 활성화해야 한다고 문서화한다.

## 공통 원칙

직접 Vault login 성공은 role, auth mount, policy, secret path가 외부적으로 동작한다는 뜻이다. Spring application이 같은 active profile과 token file path를 사용한다는 뜻은 아니다.

## 점검 방법

Vault value가 없거나 ConfigData processing 중 startup이 실패하면 다음을 비교한다.

- `spring.profiles.active`와 loaded `application-{profile}.yaml`
- active profile의 `spring.config.import`
- placeholder resolution 이후 `spring.cloud.vault.kubernetes.service-account-token-file`
- non-Kubernetes machine에서 `/var/run/secrets/kubernetes.io/serviceaccount/token` fallback이 stack trace에 보이는지

## 검증 방법

같은 active profile로 minimal Spring ConfigData probe나 context startup을 실행하고 expected Vault-derived property의 존재 여부만 boolean으로 출력한다. 별도로 Vault login과 secret-read endpoint를 호출하되 token이나 secret value는 출력하지 않는다. 이렇게 외부 Vault 접근과 Spring profile binding 문제를 분리한다.
