[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-029 — Loki Workload의 PVC 참조가 로컬 Storage Desired State와 불일치

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-04 |
| **상태** | **해결** |
| **주 담당** | **최유준 — CI/CD 및 모니터링·관측** |
| **영향 범위** | Loki Deployment, Loki Local PV/PVC, Observability Application |

## 문제 개요

Observability Application을 정상화하는 과정에서 Loki Deployment가 프로젝트에서 이미 정의해 둔 로컬 PVC와 다른 이름의 PVC를 참조하고 있는 문제가 확인됐다.

프로젝트의 Loki Storage Desired State는 다음과 같이 정의되어 있었다.

```text
loki-local-pv
→ volumeName 기반
→ loki-local-pvc
```

`loki-local-pvc`는 `storageClassName: local-storage`와 `volumeName: loki-local-pv`를 사용해 `worker-02`의 Loki Local PV를 소비하도록 구성되어 있었다.

하지만 당시 `observability/loki-deployment.yaml`은:

```yaml
persistentVolumeClaim:
  claimName: loki-storage
```

를 참조하고 있었고, 동일 Manifest 하단에는 별도의 `loki-storage` PVC 정의도 존재했다.

즉 Loki Workload가 프로젝트에서 이미 정의한 Storage Desired State를 소비하지 않고 별도의 PVC 계약을 가지고 있었다.

## 원인 분석

문제의 핵심은 PV 자체나 Loki Container의 장애가 아니라 **Storage Source of Truth와 Workload PVC 참조 사이의 Desired State 불일치**였다.

```text
프로젝트 Storage Desired State
loki-local-pv
→ loki-local-pvc

Loki Workload
→ claimName: loki-storage

결과
→ Workload와 실제 Storage 계약 불일치
```

같은 Observability 정상화 과정에서 대형 CRD 적용 문제와 Loki Configuration 오류도 함께 확인됐지만 Root Cause와 수정 계층은 각각 달랐다.

## 조치

`seokpan-gitops` PR #30에서 Loki Deployment의 PVC 참조를 다음과 같이 정정했다.

```text
Before
claimName: loki-storage

After
claimName: loki-local-pvc
```

동시에 Deployment Manifest에 중복 정의되어 있던 `loki-storage` PVC 블록을 제거했다.

현재 `main`의 Loki Deployment는 다음 계약을 사용한다.

```yaml
persistentVolumeClaim:
  claimName: loki-local-pvc
```

기존 `loki-local-pvc` 정의는:

```text
storageClassName: local-storage
volumeName: loki-local-pv
```

로 실제 Loki Local PV와 연결된다.

## 검증

PR #30 Review에서 `loki-local-pvc`가 기존 프로젝트의 Loki Local PV/PVC 정의와 일치하는 것을 확인했다.

Merge 이후 정태훈이 후속 Runtime 검증을 수행한 결과:

```text
Loki Deployment
→ 1/1

Loki Pod
→ 1/1 Running
→ Restart 0

loki-local-pvc
→ Bound

loki-local-pv
→ Bound
```

상태를 확인했다.

현재 Loki Deployment의 PVC 참조와 실제 Runtime Storage Binding이 일치한다.

## Before → Change → After

```text
Before
프로젝트 Storage
→ loki-local-pv / loki-local-pvc

Loki Workload
→ claimName: loki-storage

→ Desired State 계약 불일치

Change
claimName을 loki-local-pvc로 정정
+ 중복 loki-storage PVC 정의 제거

After
Loki Deployment 1/1
Loki Pod Running / Restart 0
loki-local-pvc Bound
→ loki-local-pv와 정상 연결
```

## 관련 사건

- [TS-005 — NGINX Gateway Fabric 대형 CRD가 클라이언트 방식 적용(client-side apply) Annotation 제한에 걸림](TS-005_NGINX_Gateway_Fabric_대형_CRD_적용_실패.md)
  - Argo CD 적용 방식 문제로 본 Storage 계약 문제와 별도 Root Cause
- Docs Issue #74 — Loki compactor CONFIG ERROR
  - 같은 PR #30에서 수정됐지만 Loki 내부 Configuration 문제로 별도 Root Cause

## 관련 근거

- Docs Issue #73: https://github.com/seokpan/seokpan-docs/issues/73
- seokpan-gitops Issue #28: https://github.com/seokpan/seokpan-gitops/issues/28
- seokpan-gitops PR #30: https://github.com/seokpan/seokpan-gitops/pull/30
- PR #30 Merge Commit: `cefa8013993dc56b49b692ff272a2d5e32d0710b`
- 현재 `observability/loki-deployment.yaml`
- 현재 `observability/loki-local-pv.yaml`
