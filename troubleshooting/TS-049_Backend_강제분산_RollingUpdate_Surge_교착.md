[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-049 — Backend 2 Replica 강제 분산과 기본 RollingUpdate가 충돌해 Rollout이 교착된 문제

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-15 |
| **상태** | **해결** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | Backend Kubernetes Deployment, 2 Replica RollingUpdate |

## 최초 문제

Backend를 2 Replica로 확장하고 두 Pod를 서로 다른 Worker에 강제 분산한 뒤 Deployment Rollout이 진행되지 않는 문제가 발생했다.

기존 두 Pod는 worker-01과 worker-02에서 모두 Ready 상태를 유지하고 있었지만, 기본 RollingUpdate는 기존 Pod를 내리기 전에 새로운 Pod를 먼저 추가하려 했다.

Backend에는 다음 강제 분산 조건이 적용돼 있었다.

```text
replicas: 2
Worker: 2대
topologyKey: kubernetes.io/hostname
whenUnsatisfiable: DoNotSchedule
```

두 Worker에 기존 Pod가 하나씩 이미 배치된 상태에서 세 번째 Surge Pod가 생성되자 분산 조건을 만족할 수 있는 Node가 없어 새 Pod가 배치되지 못했고 Rollout이 교착됐다.

기존 두 Pod는 Ready 상태였기 때문에 이 문제를 서비스 전체 중단으로 기록하지 않는다.

## 원인

문제는 2대의 Worker에 2개 Pod를 강제로 한 개씩 배치하는 조건과 기본 RollingUpdate의 Surge 방식이 함께 사용된 것이었다.

```text
Backend Pod 2개
→ worker-01 / worker-02 점유

RollingUpdate
→ 새 Pod를 먼저 Surge

DoNotSchedule
→ 같은 Worker에 추가 배치 금지

결과
→ 세 번째 Pod 배치 불가
→ Rollout 진행 불가
```

PDB `minAvailable: 1`은 Eviction API를 통한 계획된 축출을 제한하는 설정이며, Deployment RollingUpdate에서 Surge Pod를 배치하지 못한 직접 원인은 아니었다.

## 조치

Backend Deployment의 RollingUpdate 전략을 명시적으로 다음 값으로 고정했다.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 0
    maxUnavailable: 1
```

`maxSurge: 0`으로 세 번째 Pod를 먼저 만들지 않고 기존 Pod 한 개를 먼저 종료해 Worker에 배치 공간을 확보하도록 했다.

`maxUnavailable: 1`로 교체 중에도 최소 한 개의 Backend Pod가 남도록 했다.

다음 조건은 그대로 유지했다.

- Backend Replica 2개
- hostname 기준 `DoNotSchedule` 강제 분산
- PDB `minAvailable: 1`
- 기존 Backend Image·Secret·ConfigMap·Service 계약

Live Deployment를 직접 Patch하거나 Pod를 수동 삭제하는 방식으로 우회하지 않고 GitOps Desired State를 수정했다.

## 검증

병합 전 Kustomize Render와 Kubernetes server-side dry-run으로 다음 계약을 확인했다.

```text
replicas: 2
maxSurge: 0
maxUnavailable: 1
PDB minAvailable: 1
hostname DoNotSchedule
```

후속 Runtime 검증에서 Backend는 다시 다음 상태로 동작했다.

```text
Backend Replica   2/2 Ready
Restart           0 / 0
Placement         worker-01 / worker-02
Argo CD           Synced / Healthy
```

이후 Backend Image Digest를 변경한 Runtime 반영에서도 두 Replica가 새 Digest로 Ready 상태에 수렴하고 worker-01/worker-02 분산을 유지한 것을 확인했다.

따라서 2 Worker 강제 분산 상태에서 이후 Rollout이 같은 교착 조건에 다시 막히지 않고 완료되는 것을 실제 Runtime에서 확인했다.

## Before → After

```text
Before
2 Worker에 Backend Pod 2개 강제 분산
+ RollingUpdate가 세 번째 Surge Pod를 먼저 생성
→ DoNotSchedule 때문에 배치할 Node 없음
→ Rollout 교착

After
maxSurge: 0
+ maxUnavailable: 1
→ 기존 Pod 한 개를 먼저 교체
→ 새 Pod가 빈 Worker에 배치
→ 2 Replica / Worker 분산 상태로 Rollout 수렴
```

## 관련 근거

- GitOps PR #94 — Backend 2 Replica 공유 Runtime: https://github.com/seokpan/seokpan-gitops/pull/94
- GitOps PR #95 — RollingUpdate 교착 보완: https://github.com/seokpan/seokpan-gitops/pull/95
- PR #95 Merge Commit: `c6c4263b841c4f10e8bb491f02c3a4908fc90294`
- 후속 Runtime Evidence — GitOps Issue #91: https://github.com/seokpan/seokpan-gitops/issues/91
