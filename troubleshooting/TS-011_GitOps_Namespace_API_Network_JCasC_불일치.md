[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-011 — GitOps 초안이 실제 Namespace·API·Network·Jenkins Configuration as Code(JCasC, Jenkins 설정 코드화) 기준과 여러 지점에서 어긋남

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-31 ~ 2026-09-01 |
| **상태** | **코드 정합화 해결 · 일부 실제 Scrape 검증은 후속** |
| **주 담당** | **최유준 — CI/CD 및 모니터링·관측** |
| **영향 범위** | 정태훈(K8s Namespace/RBAC), 이유빈(네트워크), 김상희(DB Exporter) |

## 문제 개요

초기 `seokpan-gitops` 초안에는 임시/구버전 기준이 섞여 있었다.

대표적으로:

- `jenkins` Namespace
- `monitoring` Namespace
- 중복 Namespace 선언 파일
- Application ServiceMonitor Placeholder
- 잘못된 MariaDB/LB Exporter 주소
- Core/v1 Endpoints
- 잘못된 kubelet NetworkPolicy CIDR
- MariaDB 역할을 Primary/Replica로 정적 표기

## 1차 정합화

Namespace의 공식 선언 위치를:

```text
platform/namespaces-rbac/namespaces.yaml
```

하나로 통일했다.

공식 Namespace:

- `application`
- `platform`
- `cicd`
- `observability`
- `storage-infra`

구버전:

```text
jenkins → cicd
monitoring → observability
```

로 변경했다.

## Network/Observability 수정

- MariaDB/LB Exporter 실제 IP 정합화
- `Endpoints` → `Kubernetes EndpointSlice(서비스 Endpoint 정보를 담는 리소스)`
- Kubelet 대상 잘못된 `192.168.54.0/24` 제거
- 실제 CP/Worker `/32` 주소로 제한
- LB/VRouter 필요한 대상 추가
- MariaDB 설명은 고정 Primary/Replica가 아니라 Host + 검증 시점 실제 실행 환경(Runtime) Role로 변경

## 서버 측 검증

`kubectl apply --dry-run=server`에서 대부분의 리소스가 통과했다.

ServiceMonitor만 당시 Prometheus Operator CRD가 아직 없어서:

```text
no matches for kind "ServiceMonitor"
```

가 발생했으며, 이것은 **해당 시점의 예상된 제한사항**으로 분리했다.

## 2차 문제: Metadata는 바꿨는데 JCasC 내부 문자열은 `jenkins` 그대로

후속 PR 검증에서 Jenkins JCasC 내부의:

- Kubernetes Cloud Namespace
- Jenkins URL
- Agent Template Namespace

등이 여전히 `jenkins`를 참조하고 있음이 발견됐다.

즉 YAML Metadata와 내부 Configuration String의 정합화 범위가 달랐다.

이를 `cicd`로 다시 맞췄다.

## 3차 문제: EndpointSlice를 만들었지만 ServiceMonitor가 EndpointSlice Discovery를 사용하지 않음

외부 VM Exporter를:

```text
Service
+ EndpointSlice
+ ServiceMonitor
```

조합으로 구성했는데 리뷰에서:

```yaml
spec:
  serviceDiscoveryRole: EndpointSlice
```

가 빠져 있는 것이 발견됐다.

이 상태에서는 EndpointSlice가 존재해도 Prometheus Operator의 ServiceMonitor Discovery가 그 대상을 실제 Scrape Target으로 잡지 못할 가능성이 있었다.

## 조치

- Service ↔ EndpointSlice Label 일치
- ServiceMonitor Selector ↔ Service Label 일치
- `serviceDiscoveryRole: EndpointSlice` 반영
- Port 이름 정합화

리뷰 후 승인됐다.

## 남은 제한사항

당시 ServiceMonitor CRD 자체가 아직 실제 실행 환경에 없던 시점의 검증은 Server Dry-run으로 완결할 수 없었다. 따라서 **Manifest 정합화 완료와 실제 Prometheus Scrape 성공을 같은 완료로 표현하지 않는다.**

## 관련 근거

- 초기 GitOps 초안 PR #9: https://github.com/seokpan/seokpan-gitops/pull/9
- Namespace/API 정합화 PR #11: https://github.com/seokpan/seokpan-gitops/pull/11
- JCasC/Exporter 후속 PR #12: https://github.com/seokpan/seokpan-gitops/pull/12
- Infra Jenkins Secret Namespace 후속 PR #76: https://github.com/seokpan/seokpan-infra/pull/76