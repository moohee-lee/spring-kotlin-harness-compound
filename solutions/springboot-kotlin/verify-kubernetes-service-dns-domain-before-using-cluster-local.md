---
title: verify-kubernetes-service-dns-domain-before-using-cluster-local
tags: [springboot-kotlin, kubernetes, dns, service-discovery]
scope: cross-project
status: draft
---

# verify-kubernetes-service-dns-domain-before-using-cluster-local

## Context
Spring Boot services running in Kubernetes may fail during datasource, Kafka,
or other client initialization with `UnknownHostException` for a service FQDN.
This often happens in managed or private clusters where the cluster DNS domain
is not the Kubernetes default `cluster.local`.

## Wrong Direction
Assuming `<service>.<namespace>.svc.cluster.local` is always valid, or changing
JDBC/client settings before verifying the actual Service object and pod DNS
search domains.

## Correct Pattern
Check the target Service name in the namespace first, then check `/etc/resolv.conf`
from a running pod in the caller namespace. Build FQDNs from the real cluster
domain shown there, or use a name form that the runtime resolver is proven to
resolve in that cluster. For CloudNativePG, prefer the generated service names
such as `<cluster>-rw` for primary writes and `<cluster>-ro` for replica reads.

## Reusable Insight
`UnknownHostException` is a DNS-resolution failure, not a PostgreSQL credential,
schema, or network-policy failure. A Service can exist and still fail if the
configured hostname uses the wrong service name or wrong cluster DNS suffix.

## Detection
Look for hard-coded `.svc.cluster.local` in Kubernetes profile configuration or
ConfigMaps. Compare it against `kubectl get svc -n <namespace>` and the pod
resolver search domains. If all `cluster.local` lookups return NXDOMAIN while
`svc.<actual-cluster-domain>` resolves, the suffix is wrong.

## Verification
Run DNS and TCP checks from a pod in the caller namespace:

```bash
kubectl exec -n <caller-namespace> <pod> -- cat /etc/resolv.conf
kubectl exec -n <caller-namespace> <pod> -- nslookup <service>.<target-namespace>.svc.<cluster-domain>
kubectl exec -n <caller-namespace> <pod> -- nc -vz -w 3 <service>.<target-namespace>.svc.<cluster-domain> 5432
```
