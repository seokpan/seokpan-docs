[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-011 — GitOps 초안이 실제 Namespace·API·Network·Jenkins Configuration as Code(JCasC, Jenkins 설정 코드화) 기준과 여러 지점에서 어긋남

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-31 ~ 2026-09-01 |
| **상태** | **코드 정합화 해결 · 일부 실제 Scrape 검증은 후속** |
| **주 담당** | **최유준 — CI/CD 및 모니터링·관측** |
| **영향 범위** | 정태훈(K8s Namespace/RBAC), 이유빈(Network), 김상희(DB Exporter) |

## 문제 개요

`seokpan-gitops` 초안은 방향 자체는 맞았지만 실제 프로젝트와 여러 곳에서 어긋나 있었다.

대표 문제:

```text
문서 Namespace: cicd / observability
초안: jenkins / monitoring

Gateway:
초안 LoadBalancer
실제 정책 NodePort

HAProxy:
TLS passthrough 필요
초안에 NodePort backend 연결 없음

Harbor:
초안 FQDN이 공식값과 다름

DB:
Exporter endpoint 없음
```

## 1차 정합화

Issue #5에서 문서+구현 전체 기준으로 수정했다.

## Network/Observability 수정

NGF Service가:

```text
LoadBalancer
```

이면 외부 LB가 이미 HAProxy인 프로젝트 구조와 충돌한다.

따라서:

```text
NodePort
externalTrafficPolicy: Cluster
30080 / 30443
```

로 정리했다.

HAProxy는:

```text
VIP:443
→ worker-01:30443
→ worker-02:30443
```

TCP passthrough 구조로 반영했다.

## 서버 측 검증

단순 YAML 파싱에서 끝내지 않고 실제 Kubernetes API에:

```bash
kubectl apply --dry-run=server
```

로 검증했다.

## 2차 문제: Metadata는 바꿨는데 JCasC 내부 문자열은 `jenkins` 그대로

PR #12에서:

```yaml
metadata:
  namespace: cicd
```

로 바꿨지만 JCasC 문자열 내부의:

```text
namespace: "jenkins"
```

가 남아 있었다.

즉 **YAML Metadata 정합화와 Embedded Configuration 정합화는 별도**였다.

후속 수정에서:

```text
jenkins
→ cicd
```

로 일치시켰다.

## 3차 문제: Kubernetes EndpointSlice(서비스 Endpoint 정보를 담는 리소스)를 만들었지만 ServiceMonitor가 EndpointSlice Discovery를 사용하지 않음

MariaDB/MaxScale/Harbor 외부 VM을 관측하기 위해 Kubernetes EndpointSlice 리소스를 생성했다.

하지만 초기 ServiceMonitor에는:

```yaml
spec:
  serviceDiscoveryRole: EndpointSlice
```

가 없었다.

Prometheus Operator 기본 방식과 실제 EndpointSlice 연결 방식이 다를 수 있으므로 실제 Scrape로 이어지지 않을 수 있었다.

후속 수정에서 `serviceDiscoveryRole: EndpointSlice`를 추가했다.

## 조치

- Namespace 정합화
- Gateway NodePort 정합화
- HAProxy 443 passthrough
- Harbor FQDN 수정
- Redis/PV 분리
- DB Exporter endpoint
- JCasC 내부 Namespace 수정
- EndpointSlice ServiceMonitor Discovery Role 추가

## 남은 제한사항

**Manifest의 server dry-run 성공 = 실제 Prometheus Scrape 성공은 아니다.**

따라서 실제:

```text
Prometheus Target UP
Grafana Query 성공
Loki Log 조회 성공
```

등은 별도 실제 클러스터 실행 환경(Runtime) 검증이 필요하다.

## 관련 근거

- GitOps 추적 Issue #5: https://github.com/seokpan/seokpan-gitops/issues/5
- PR #12: https://github.com/seokpan/seokpan-gitops/pull/12
