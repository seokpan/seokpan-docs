[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-049 — Backend 2 Replica 강제 분산과 RollingUpdate가 충돌해 Rollout이 교착된 문제

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-15 |
| **상태** | **해결** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | Backend Deployment, 2 Replica RollingUpdate |

## 최초 문제

Backend를 2 Replica로 늘리고 각 Pod를 서로 다른 Worker에 하나씩 배치한 뒤, Deployment를 새 설정으로 갱신하는 과정에서 Rollout이 더 이상 진행되지 않았다.

당시 Backend Pod 두 개는 각각 worker-01과 worker-02에서 Ready 상태였다.

```text
worker-01 → Backend Pod 1개
worker-02 → Backend Pod 1개
```

Backend에는 같은 Worker에 두 Pod가 배치되지 않도록 다음 설정이 적용돼 있었다.

```text
topologyKey: kubernetes.io/hostname
whenUnsatisfiable: DoNotSchedule
```

기본 RollingUpdate는 기존 Pod를 먼저 없애지 않고 새 Pod를 하나 더 만들려고 했다.

하지만 Worker가 두 대뿐이고 두 Worker에 기존 Pod가 이미 하나씩 있었기 때문에 세 번째 Pod를 배치할 곳이 없었다.

새 Pod를 배치하지 못하면서 Rollout이 더 진행되지 않는 **교착 상태**가 됐다.

기존 Pod 두 개는 계속 Ready 상태였기 때문에 서비스 전체가 중단된 장애는 아니었다.

## 원인

Worker 수와 Pod 강제 분산 조건을 고려하지 않은 RollingUpdate 방식이 원인이었다.

```text
Worker 2대
+ Backend Pod 2개를 Worker마다 하나씩 강제 배치

기본 RollingUpdate
→ 기존 Pod를 유지한 채 새 Pod 하나 추가 시도

DoNotSchedule
→ 같은 Worker에 두 번째 Backend Pod 배치 금지

결과
→ 세 번째 Pod를 배치할 곳이 없음
→ Rollout 교착
```

PDB `minAvailable: 1`은 계획된 Pod 축출 때 최소 한 개를 남기기 위한 설정이다. 이번처럼 새 Pod를 배치하지 못한 직접 원인은 아니었다.

## 조치

RollingUpdate 방식을 다음과 같이 명시했다.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 0
    maxUnavailable: 1
```

`maxSurge: 0`으로 새 Pod를 먼저 하나 더 만들지 않도록 했다.

대신 기존 Pod 하나를 먼저 종료해 Worker 한 곳을 비우고, 그 자리에 새 Pod를 올리도록 했다.

`maxUnavailable: 1`을 사용해 교체 중에도 Backend Pod가 최소 한 개는 계속 실행되도록 했다.

다음 설정은 그대로 유지했다.

- Backend Replica 2개
- 서로 다른 Worker에 강제로 나눠 배치
- PDB `minAvailable: 1`
- 기존 Image·Secret·ConfigMap·Service

실행 중인 Deployment를 직접 수정하거나 Pod를 사람이 수동으로 삭제하지 않고 Git에 저장된 Deployment 설정을 변경했다.

## 검증

병합 전 Kustomize Render와 Kubernetes server-side dry-run으로 다음 값을 확인했다.

```text
replicas: 2
maxSurge: 0
maxUnavailable: 1
PDB minAvailable: 1
hostname DoNotSchedule
```

이후 실제 클러스터에서 Backend가 다음 상태가 된 것을 확인했다.

```text
Backend Replica   2/2 Ready
Restart           0 / 0
worker-01         Pod 1개
worker-02         Pod 1개
Argo CD           Synced / Healthy
```

그 뒤 Backend Image Digest를 다시 변경해 재배포했을 때에도 Rollout이 완료됐다.

새 Pod 두 개는 다시 worker-01과 worker-02에 하나씩 배치됐고 모두 Ready 상태가 됐다.

따라서 같은 2 Worker·2 Replica 구조에서도 새 설정으로 Pod를 교체할 수 있음을 확인했다.

## Before → After

```text
Before
Worker 2대에 Backend Pod 2개를 하나씩 배치
→ RollingUpdate가 세 번째 Pod를 먼저 만들려고 함
→ 배치 가능한 Worker 없음
→ Rollout 교착

After
maxSurge: 0
→ 기존 Pod 하나를 먼저 종료
→ 빈 Worker에 새 Pod 배치
→ 다음 Pod도 같은 방식으로 교체
→ 최종 2/2 Ready, Worker별 1개 배치
```

## 관련 근거

- GitOps PR #94 — Backend 2 Replica 구성: https://github.com/seokpan/seokpan-gitops/pull/94
- GitOps PR #95 — RollingUpdate 수정: https://github.com/seokpan/seokpan-gitops/pull/95
- PR #95 Merge Commit: `c6c4263b841c4f10e8bb491f02c3a4908fc90294`
- 후속 실제 클러스터 검증 — GitOps Issue #91: https://github.com/seokpan/seokpan-gitops/issues/91
