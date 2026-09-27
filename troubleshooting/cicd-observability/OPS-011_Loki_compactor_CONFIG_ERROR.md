[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-030 — Loki compactor의 `delete_request_store` 누락으로 CONFIG ERROR 발생

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-04 |
| **상태** | **해결** |
| **주 담당** | **최유준 — CI/CD 및 모니터링·관측** |
| **영향 범위** | Loki ConfigMap, Loki Pod 기동, Grafana → Loki 조회 경로 |

## 문제 개요

Observability Application 정상화 과정에서 Loki가 Configuration 오류로 정상 실행되지 못하는 문제가 확인됐다.

당시 Loki는 retention 기능을 사용하는 다음 compactor 설정을 가지고 있었다.

```yaml
compactor:
  working_directory: /loki/compactor
  retention_enabled: true
```

하지만 현재 프로젝트의 filesystem 기반 Loki 구성에서 사용하는 `compactor.delete_request_store` 값이 빠져 있었다.

`seokpan-gitops` PR #30에는 이 설정 누락으로 CONFIG ERROR가 발생했다고 기록되어 있다.

## 원인 분석

현재 프로젝트의 Loki는 저장 방식으로 filesystem을 사용한다.

```yaml
common:
  storage:
    filesystem:
```

그러나 retention을 사용하는 compactor 설정에는 삭제 요청 정보를 어디에 저장할지 지정하는 `delete_request_store`가 없었다.

```text
현재 프로젝트 Loki
→ filesystem 사용
→ retention_enabled=true
→ delete_request_store 미설정
→ CONFIG ERROR
```

따라서 문제는 PVC 참조나 대형 CRD 적용 방식이 아니라 **retention을 사용하는 Loki compactor 설정에 filesystem용 `delete_request_store`가 빠져 있었던 것**이었다.

## 조치

`observability/loki-config.yaml`에 다음 설정을 추가했다.

```yaml
compactor:
  working_directory: /loki/compactor
  delete_request_store: filesystem
  retention_enabled: true
```

현재 `main`에도 동일 설정이 유지되고 있다.

## 검증

PR #30 단계에서 수정 후 Loki Pod가 `Running(1/1)` 상태로 유지되는 것을 확인했다.

Merge 이후 정태훈이 후속 실행환경 검증을 다시 수행한 결과:

```text
Loki Deployment
→ 1/1

Loki Pod
→ 1/1 Running
→ Restart 0
```

상태를 확인했다.

최근 Loki Log에서도 PR #30에서 수정한 CONFIG ERROR의 재발을 확인하지 못했다.

추가로 NetworkPolicy상 Loki 접근이 허용된 Grafana Pod에서 다음 Loki API를 확인했다.

```text
/loki/api/v1/query_range
/loki/api/v1/labels
```

응답:

```json
{"status":"success"}
```

일반 임시 Pod에서 Loki Service 접근 시 timeout이 발생했지만 Loki ingress NetworkPolicy가 Grafana와 Alloy만 허용하고 있었으므로, 이는 Loki 장애가 아니라 의도된 접근 차단으로 판정했다.

단, `query_range`의 실제 Stream 결과는 빈 배열이었으므로 이번 검증은 **Grafana에서 Loki Query API까지 접근 가능함**까지만 증명한다. 실제 저장 Log Stream이 반환됐다고 확대해서 기록하지 않는다.

## Before → Change → After

```text
Before
filesystem 기반 Loki
+ retention_enabled=true
+ delete_request_store 미설정
→ CONFIG ERROR

Change
compactor.delete_request_store=filesystem 추가

After
Loki Pod Running / Restart 0
기존 CONFIG ERROR 재발 없음
Grafana → Loki Query API status=success
```

## 관련 사건

- [TS-029 — Loki Deployment가 프로젝트에서 정의한 PVC와 다른 PVC를 참조함](TS-029_Loki_PVC_Desired_State_불일치.md)
  - Loki Deployment가 잘못된 PVC를 참조한 문제로, 본 설정 누락과 원인이 다르다.
- [TS-005 — NGINX Gateway Fabric 대형 CRD가 클라이언트 방식 적용(client-side apply) Annotation 제한에 걸림](TS-005_NGINX_Gateway_Fabric_대형_CRD_적용_실패.md)
  - Argo CD의 대형 CRD 적용 방식 문제로, Loki compactor 설정 누락과 원인이 다르다.

## 관련 근거

- Docs Issue #74: https://github.com/seokpan/seokpan-docs/issues/74
- seokpan-gitops Issue #28: https://github.com/seokpan/seokpan-gitops/issues/28
- seokpan-gitops PR #30: https://github.com/seokpan/seokpan-gitops/pull/30
- PR #30 Merge Commit: `cefa8013993dc56b49b692ff272a2d5e32d0710b`
- 현재 `observability/loki-config.yaml`
