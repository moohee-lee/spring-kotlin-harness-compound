---
title: cluster.local 사용 전 Kubernetes 서비스 DNS 도메인을 확인한다
tags: [springboot-kotlin, kubernetes, dns, service-discovery]
scope: cross-project
status: active
principles: [configuration-ownership, runtime-boundaries]
---

# cluster.local 사용 전 Kubernetes 서비스 DNS 도메인을 확인한다

> 관련 공통 원칙: [설정 소유권과 로딩 시점](../../principles/ko/configuration-ownership.md), [런타임 경계와 컨테이너 계약](../../principles/ko/runtime-boundaries.md)

## 적용 시점

Kubernetes에서 실행되는 Spring Boot service가 datasource, Kafka, 기타 client 초기화 중 service FQDN에 대해 `UnknownHostException`으로 실패할 때 적용한다. managed/private cluster에서는 DNS domain이 Kubernetes 기본값 `cluster.local`이 아닐 수 있다.

## 피해야 할 방향

`<service>.<namespace>.svc.cluster.local`이 항상 유효하다고 가정하면 안 된다. 실제 Service object와 caller pod의 DNS search domain을 확인하기 전에 JDBC/client 설정부터 바꾸는 것도 피한다.

## 권장 패턴

먼저 target namespace의 Service 이름을 확인하고, caller namespace에서 실행 중인 pod의 `/etc/resolv.conf`를 확인한다. 거기에 드러난 실제 cluster domain으로 FQDN을 만들거나, 해당 cluster에서 resolver가 실제로 resolve하는 name form을 사용한다. CloudNativePG라면 primary write에는 `<cluster>-rw`, replica read에는 `<cluster>-ro` 같은 generated service name을 우선 확인한다.

## 공통 원칙

`UnknownHostException`은 PostgreSQL credential, schema, network policy 문제가 아니라 DNS resolution failure다. Service가 존재해도 hostname의 service name이나 cluster DNS suffix가 틀리면 실패한다.

## 점검 방법

Kubernetes profile configuration이나 ConfigMap에서 hard-coded `.svc.cluster.local`을 검색한다. `kubectl get svc -n <namespace>`와 pod resolver search domain을 비교한다. 모든 `cluster.local` lookup은 NXDOMAIN인데 `svc.<actual-cluster-domain>`은 resolve된다면 suffix가 틀린 것이다.

## 검증 방법

caller namespace의 pod에서 DNS와 TCP를 확인한다.

```bash
kubectl exec -n <caller-namespace> <pod> -- cat /etc/resolv.conf
kubectl exec -n <caller-namespace> <pod> -- nslookup <service>.<target-namespace>.svc.<cluster-domain>
kubectl exec -n <caller-namespace> <pod> -- nc -vz -w 3 <service>.<target-namespace>.svc.<cluster-domain> 5432
```
