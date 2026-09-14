# MVP 구축·자동화 Runbook
## Kubernetes & Application Integration

## 1. 목적과 사용 범위

이 문서는 「石나가는 판단」 1차 프로젝트의 Kubernetes & Application Integration 영역을 실제로 구축·적용·확인·재실행·복구·Rollback하기 위한 실행 Runbook이다.

09 문서가 책임·Provider/Consumer·Integration Gate를 정의한다면, 본 문서는 각 Gate를 실제 작업으로 통과하는 절차를 소유한다.

```text
09 = What / Why / Responsibility / Gate
11 = Pre-check / Apply / Verify / Re-run / Recovery / Rollback
12 = Test Case / Measurement / PASS·FAIL / Evidence
```

본 문서는 현재 구현 상태와 Repository 자산을 기준으로 작성한다. 구현되지 않은 HPA, Application HTTPRoute 등은 실행 완료 절차처럼 쓰지 않고 구현 전제와 실행 순서를 구분한다.

직접 기준:

- [`09_MVP_실행·통합_실시설계_Kubernetes_Application_Integration_정태훈.md`](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Kubernetes_Application_Integration_정태훈.md)
- [`10_GitHub_협업_및_Repository_운영.md`](../10_GitHub_협업_및_Repository_운영.md)
- `PROJECT_CHANGES.md`
- `MVP_IMPLEMENTATION_BASELINE.md`
- `seokpan-app` 현재 `main`, Issue, PR
- `seokpan-gitops` 현재 `main`, Issue, PR
- `seokpan-infra` 현재 `main`, Issue, PR
- 실제 Runtime Evidence

## 2. 실행 원칙

### 2.1 GitOps 우선

Argo CD가 관리하는 Kubernetes Desired State는 Git 변경을 기본 운영 경로로 사용한다.

```text
Issue
→ Branch
→ Desired State 변경
→ 정적 검증
→ PR
→ Review
→ Merge
→ Argo CD Sync
→ Kubernetes 반영
```

운영 상태를 맞추기 위해 `kubectl edit`, 직접 `kubectl apply`, 직접 Scale을 기본 방식으로 사용하지 않는다. 직접 변경은 진단 또는 명시적인 검증 절차에서만 사용하고, Git Desired State와 충돌하면 Argo CD `selfHeal`에 의해 원복될 수 있음을 전제로 한다.

### 2.2 상태 판정

다음을 같은 의미로 취급하지 않는다.

```text
Implemented ≠ Merged ≠ Running ≠ Validated
Image Pushed ≠ GitOps Updated ≠ Argo Synced ≠ Pod Ready
Provider Ready ≠ Consumer Wired ≠ Integration Validated
```

### 2.3 Secret 비노출

Runbook 실행 중 Secret의 존재와 Key 구조는 확인할 수 있으나 Password, 전체 DB URL, Token, Private Key를 콘솔·Issue·PR·문서에 출력하거나 기록하지 않는다.

### 2.4 중단 우선

아래 조건에서는 다음 단계로 진행하지 않는다.

- 현재 Release Image Digest 미확정
- 필요한 Secret/ConfigMap 부재
- Active Migration Job 존재
- Migration 실패 또는 후검증 미완료
- Backend 1 Replica First Runtime 실패
- Provider readiness 실패
- GitOps Desired State와 실제 작업 대상 Revision 불일치

## 3. 현재 실행 기준 상태

현재 기준 상태는 다음과 같다.

| 영역 | 상태 |
| --- | --- |
| Kubernetes Cluster / Calico / CoreDNS | Validated |
| Namespace / RBAC / Argo CD | Validated |
| `apps-backend`, `apps-frontend` Child Application | Root 편입 완료 |
| Backend Deployment | `replicas: 0`, `git-pending` |
| Frontend Deployment | `replicas: 0`, `git-pending` |
| Redis Runtime / Persistence | Validated |
| Backend → Redis | Not Tested |
| DB Runtime/Migration Secret 공급 구조 | Implemented / Validated |
| One-shot Migration 자산 | Implemented / Merged / Static+API Validated |
| 실제 Migration | Not Tested |
| Application HTTPRoute | Not Implemented |
| HPA | Not Implemented |

이 표는 12의 공식 PASS/FAIL Evidence를 대체하지 않는다.

## 4. 공통 Pre-check

### 4.1 Repository 최신 상태

작업 시작 전에 관련 Repository가 의도한 `main` 기준인지 확인한다.

```bash
git status
git branch --show-current
git fetch --prune
git log --oneline --decorate -n 5
```

필요한 작업은 별도 Branch에서 수행한다.

### 4.2 Kubernetes Node

```bash
kubectl get nodes -o wide
```

최소 확인:

- Control Plane 3대
- Worker 2대
- 대상 Node `Ready`

### 4.3 Argo CD Application

```bash
kubectl -n argocd get applications

kubectl -n argocd get application \
  apps-backend apps-frontend gateway redis
```

Backend/Frontend Runtime 활성화 전에 `apps-backend`, `apps-frontend`가 존재하고 의도한 `main` Revision을 감시하는지 확인한다.

### 4.4 Application Namespace

```bash
kubectl -n application get deploy,svc
```

현재 Runtime 활성화 전에는 Backend/Frontend가 `0/0`이어도 정상이다.

### 4.5 Redis

```bash
kubectl -n platform get statefulset,pod,svc,pvc
```

Redis의 실제 Backend Consumer 연결 여부는 Backend Runtime 단계에서 별도 확인한다.

### 4.6 Secret / CA 존재 확인

값을 출력하지 않고 Metadata와 Key 이름만 확인한다.

```bash
kubectl -n application get secret \
  backend-db-runtime \
  backend-db-migration

kubectl -n application get configmap \
  seokpan-internal-ca
```

필요 Key:

```text
backend-db-runtime
├─ SEOKPAN_IDENTITY_DATABASE_URL
└─ SEOKPAN_GAME_DATABASE_URL

backend-db-migration
└─ SEOKPAN_MIGRATION_DATABASE_URL

seokpan-internal-ca
└─ ca.crt
```

Secret 값의 공급 Source of Truth는 `seokpan-infra` Ansible + Vault다.

## 5. Application Artifact 준비

### 5.1 Release Source 확인

A-09에서 Image Pipeline Capability가 검증되었더라도 A-10 Source가 변경되면 현재 Runtime이 사용할 Image는 다시 생성·검증해야 한다.

```text
A-10 Source Merge
→ Jenkins main Pipeline
→ Build
→ Harbor Push
→ Scan / Acceptance
→ Digest 확인
→ Runtime Candidate 확정
```

과거 A-09 Artifact를 현재 A-10 Artifact로 간주하지 않는다.

### 5.2 Image 정책

실제 Runtime 활성화에는 이동 가능한 `latest` 또는 `git-pending`을 사용하지 않는다.

목표 형식:

```text
harbor.seokpan.soldesk.store/seokpan/backend@sha256:<verified-digest>
harbor.seokpan.soldesk.store/seokpan/frontend@sha256:<verified-digest>
```

Digest 값 자체는 실행 시점의 검증된 Artifact에서 가져온다.

## 6. DB / CA / Provider Pre-check

### 6.1 DB Endpoint

Backend와 Migration의 공식 DB Endpoint는 다음 계약을 사용한다.

```text
db.seokpan.soldesk.store:3306
```

Runtime Credential과 Migration Credential을 혼용하지 않는다.

### 6.2 CA

공개 CA 계약:

```text
ConfigMap: seokpan-internal-ca
Key:       ca.crt
Mount:     /etc/seokpan/pki/ca.crt
Env:       SEOKPAN_DATABASE_CA_FILE=/etc/seokpan/pki/ca.crt
```

CA 자체 검증과 실제 Backend→MaxScale TLS 연결 성공은 서로 다른 결과로 본다.

### 6.3 Migration 선행조건

Mutation Migration 전에 최소 다음이 충족되어야 한다.

- 승인된 Backend Image Digest
- `backend-db-migration` Secret 준비
- DB 상태 확인
- Replication 상태 확인
- Backup / Restore 가능 상태 확인
- CA 준비
- Active Migration Job 없음
- 실행 승인 Reference 존재

## 7. One-shot Migration Runbook

Migration 자산은 `seokpan-gitops/apps/backend/migration/`에 있으며 일반 Backend Kustomization과 분리되어 있다. Argo CD Auto-Sync로 DDL을 실행하지 않는다.

### 7.1 Active Job Guard

```bash
kubectl get jobs -n application \
  -l app.kubernetes.io/name=backend-db-migration \
  -o custom-columns='NAME:.metadata.name,ACTIVE:.status.active,SUCCEEDED:.status.succeeded,FAILED:.status.failed'
```

Active Migration Job이 있으면 새 Mutation Job을 생성하지 않는다.

### 7.2 Read-only current Render

```bash
python3 apps/backend/migration/render-job.py \
  current \
  --image 'harbor.seokpan.soldesk.store/seokpan/backend@sha256:<verified-digest>' \
  --output /tmp/backend-migration-current.yaml
```

### 7.3 Mutation Render

지원 Mutation:

```text
stamp-baseline
upgrade-head
```

예:

```bash
python3 apps/backend/migration/render-job.py \
  upgrade-head \
  --image 'harbor.seokpan.soldesk.store/seokpan/backend@sha256:<verified-digest>' \
  --approval-ref 'seokpan/<repo>#<issue-number>:issuecomment-<comment-id>' \
  --output /tmp/backend-migration-upgrade.yaml
```

Approval Reference에는 Credential을 기록하지 않는다.

### 7.4 Kubernetes API Dry-run

```bash
kubectl create --dry-run=server \
  -f /tmp/backend-migration-upgrade.yaml
```

Dry-run 성공은 실제 DB Migration 성공을 의미하지 않는다.

### 7.5 실제 실행

모든 Gate가 충족된 경우에만 새 Job을 생성한다.

```bash
kubectl create \
  -f /tmp/backend-migration-upgrade.yaml
```

완료·실패한 기존 Job에 `kubectl apply`하여 재사용하지 않는다.

### 7.6 실행 후 확인

최소 확인 대상:

```text
Job terminal status
Alembic current
Replication
MaxScale Read/Write
기존 데이터 보존
```

구체 PASS/FAIL 기준과 Evidence 형식은 12에서 관리한다.

### 7.7 실패

```text
Migration Failed
→ Backend Runtime 활성화 중단
→ Job / Log 확인
→ DB 상태 확인
→ Replication 확인
→ Rollback / Restore 필요성 판단
→ 원인 수정
→ 새로운 승인
→ 새로운 Job 생성
```

Schema 변경은 Git Revert만으로 DB를 원상복구할 수 있다고 가정하지 않는다.

## 8. Backend 1 Replica First Runtime

Backend 1 Replica는 실제 Provider Integration의 첫 Runtime Gate다.

### 8.1 선행조건

- A-10 Provider wiring 완료
- 현재 `main` 기준 Backend Image Digest 확보
- Migration 완료 및 후검증
- `backend-db-runtime` Secret 준비
- `seokpan-internal-ca` 준비
- Redis Runtime 준비
- `apps-backend` Argo CD Application 존재

### 8.2 GitOps 변경

현재 주요 변경 대상:

```text
apps/backend/kustomization.yaml
apps/backend/deployment.yaml
```

현재 상태:

```text
image: git-pending
replicas: 0
```

변경 목표:

```text
verified Backend Digest
replicas: 1
```

실제 변경은 Branch → PR → Review → Merge로 수행한다.

### 8.3 Argo 반영 확인

Merge 후:

```bash
kubectl -n argocd get application apps-backend

kubectl -n application get deployment backend

kubectl -n application get pods \
  -l app.kubernetes.io/name=backend -o wide

kubectl -n application rollout status \
  deployment/backend
```

### 8.4 Probe

Deployment 기준 Health Endpoint:

```text
Startup   /health/startup
Liveness  /health/live
Readiness /health/ready
```

Probe 성공과 DB/Redis Provider E2E 성공을 동일하게 보지 않는다.

### 8.5 Provider 확인

Backend Pod 기동 후 최소 확인 범위:

```text
Backend → MaxScale TLS
Backend → Identity DB
Backend → Game DB
Backend → Redis Service DNS
Readiness Provider 상태
```

구체 Test Case와 Evidence는 12에서 관리한다.

### 8.6 실패 시

신규 Image 또는 Runtime 변경으로 Backend가 정상화되지 않으면 Cluster를 직접 수정하지 않고 GitOps Revision을 되돌린다.

```text
문제 Revision 식별
→ Git Revert 또는 후속 수정 PR
→ Review / Merge
→ Argo CD Sync
→ 이전 Desired State 회복 확인
```

첫 Runtime 이전 상태로 복귀해야 하는 경우 `replicas: 0`과 이전 안전 Image 상태를 명시적으로 복구한다.

## 9. Backend 2 Replica Scale-out

1 Replica 통합 검증 전에는 2 Replica로 확장하지 않는다.

### 9.1 선행조건

- Backend 1 Replica Running
- DB/Redis Provider 연결 정상
- Migration 후검증 완료
- Realtime/Runner가 Process-local Authority에 의존하지 않는 A-10 상태

### 9.2 GitOps Scale-out

`apps/backend/deployment.yaml`의 Desired Replica를 2로 변경하고 PR/Merge한다.

Merge 후:

```bash
kubectl -n application get deployment backend

kubectl -n application get pods \
  -l app.kubernetes.io/name=backend -o wide

kubectl -n application rollout status \
  deployment/backend
```

### 9.3 확인 범위

```text
Pod 2개 Ready
Worker 배치
Redis 공유 Runtime State
Redis Pub/Sub 기반 변경 알림
Runner 중복 실행 방지
Reconnect 수렴
Cross-replica Realtime
```

측정 방법과 공식 PASS/FAIL은 12에서 관리한다.

### 9.4 실패 시 축소

2 Replica에서만 문제가 발생하면 원인 분석 동안 GitOps를 통해 1 Replica 정상 기준으로 복귀할 수 있다.

```text
replicas: 2 문제
→ Git Revert / 수정 PR
→ replicas: 1
→ Argo CD Sync
→ 1 Replica 정상 상태 확인
```

## 10. HPA / Workload Distribution

현재 `main` 기준 Backend HPA와 Metrics Server 연계 Application 자산은 구현 완료 상태로 확인되지 않는다. 따라서 아래는 구현·적용 순서를 정의하는 Runbook 경계다.

### 10.1 선행조건

- Backend 1 Replica 통합 성공
- 가능하면 2 Replica 수동 Scale-out 경로 확인
- Container Resource Request/Limit 결정
- Metrics Server 또는 Resource Metrics 경로 확인

### 10.2 구현 순서

```text
Resource Request/Limit 정의
→ Resource Metrics 경로 준비
→ HPA Desired State 작성
→ Kustomize 포함
→ 정적/API 검증
→ GitOps PR
→ Merge
→ Argo CD Sync
→ HPA 상태 확인
```

CPU/Memory Target, `minReplicas`, `maxReplicas`는 근거 없이 본 문서에서 임의 확정하지 않는다.

### 10.3 Runtime 확인

구현 후 확인 명령 예:

```bash
kubectl -n application get hpa
kubectl -n application describe hpa backend
kubectl top pods -n application
kubectl top nodes
```

해당 명령은 Metrics API가 준비된 이후에만 유효하다.

### 10.4 Rollback

HPA 자체가 Runtime 불안정을 만든 경우 Git에서 HPA Desired State를 제거 또는 이전 정상 정책으로 Revert하고 Argo CD로 반영한다. 수동 Scale로 Git Desired State와 경쟁하지 않는다.

## 11. Frontend Runtime

### 11.1 현재 상태

현재 Frontend Desired State는 존재하지만 Runtime은 비활성이다.

```text
replicas: 0
image: git-pending
apps-frontend Child Application: 연결 완료
```

### 11.2 선행조건

- 실제 Frontend Image Digest 확보
- Container Runtime Smoke 확인
- `apps-frontend` Argo CD Application 확인
- Backend Runtime과 동일 Origin 계약 준비

Frontend는 환경별 Backend 절대 URL을 Image에 굽지 않고 같은 Origin의 `/api/v1`, `/ws/v1`을 사용한다.

### 11.3 GitOps 변경

주요 대상:

```text
apps/frontend/kustomization.yaml
apps/frontend/deployment.yaml
```

실제 Runtime Replica 수는 해당 활성화 PR에서 확정한다. 현재 문서에서 임의 수치를 고정하지 않는다.

### 11.4 Merge 후 확인

```bash
kubectl -n argocd get application apps-frontend

kubectl -n application get deployment frontend
kubectl -n application get service frontend

kubectl -n application get pods \
  -l app.kubernetes.io/name=frontend -o wide
```

## 12. Application HTTPRoute / Gateway Integration

### 12.1 현재 경계

Gateway Platform은 구성되어 있으나 Application HTTPRoute는 현재 `main`에서 확인되지 않는다.

목표 Route 계약:

```text
/       → frontend:8080
/api/v1 → backend:8000
/ws/v1  → backend:8000
```

### 12.2 선행조건

- Frontend Service Ready
- Backend Service Ready
- Gateway Platform Synced/Healthy
- TLS/Hostname Platform 경로 정상

### 12.3 구현 순서

```text
Application HTTPRoute Manifest 작성
→ Backend/Frontend Service 참조 확인
→ Hostname / Path Rule 확인
→ Kustomize 포함
→ server-side dry-run
→ GitOps PR
→ Merge
→ Argo CD Sync
→ HTTPRoute Conditions 확인
→ HTTPS/WSS Runtime 확인
```

### 12.4 적용 후 확인 예

```bash
kubectl get gateway -A
kubectl get httproute -A
kubectl describe httproute -n application <route-name>
```

실제 Route 이름은 구현 시 Manifest 기준을 사용한다.

### 12.5 실패 시

Route 변경으로 외부 경로가 깨지면 Git Revert로 이전 Route Desired State를 복구한다. Gateway Platform 자체가 정상인 경우 불필요하게 GatewayClass/Gateway를 재설치하지 않는다.

## 13. Argo CD 운영

### 13.1 Desired / Live 확인

```bash
kubectl -n argocd get applications
```

Application별로 최소 다음을 확인한다.

```text
Target Revision
Sync Status
Health Status
```

### 13.2 Self-Heal 원칙

`apps-backend`, `apps-frontend`는 automated sync와 `selfHeal: true` 경로에 편입되어 있다. Live Resource를 직접 수정하여 지속 운영 상태를 만들지 않는다.

### 13.3 Rollback 원칙

프로젝트의 기본 GitOps Rollback은 다음 흐름이다.

```text
정상 Revision 식별
→ Git Revert
→ PR
→ Review
→ Merge
→ Argo CD Sync
→ Kubernetes 복구
```

Argo CD UI에서 임의 Revision을 장기 운영 기준으로 만들지 않고 Git `main`과 Desired State를 다시 일치시킨다.

## 14. Application Observability 연결

Observability Platform 자체의 구축·운영은 Delivery / Observability 담당 영역이다. Kubernetes & Application Integration은 Application Runtime이 관측 대상이 될 수 있도록 계약을 제공한다.

### 14.1 현재 자산과 상태

현재 `seokpan-gitops`에는 다음 pending 자산이 존재한다.

```text
observability/servicemonitor-app.yaml.pending
```

이 파일은 Argo CD 동기화 대상에서 제외된 상태다. 현재 파일에는 다음 값이 들어 있다.

```text
Namespace: application
Service Port: http
Metrics Path: /metrics
```

다만 selector는 아직 과거 가정값인 `app.kubernetes.io/part-of: seokpan`을 사용한다. 현재 실제 Backend Service selector는 `app.kubernetes.io/name: backend`이므로 `.pending` 확장자만 제거해서는 안 된다.

또한 Backend가 실제 `/metrics`를 제공하는지와 Application Runtime이 활성화되어 있는지를 별도로 확인해야 한다.

관련 Current-State 정합화는 `seokpan-gitops#86`에서 추적한다.

### 14.2 활성화 순서

```text
Backend Runtime Running
→ Backend Service 실제 Label 확인
→ Backend /metrics 제공 여부 확인
→ servicemonitor-app selector 정합화
→ Service Port http 확인
→ pending 제거 / 정식 YAML 편입
→ GitOps PR
→ Delivery / Observability 담당 Review
→ Merge
→ Argo CD Sync
→ Prometheus Target 확인
```

Observability Platform이 Running이라는 사실만으로 Application Observability Integration을 완료로 판정하지 않는다.

### 14.3 역할 경계

Kubernetes & Application Integration은 다음을 확인한다.

```text
Health Endpoint
Application Metrics Endpoint
Backend Service Label / Port
Container / Application Log
Pod / Replica Identity
Deployment / Service Metadata
```

Prometheus Target·Metric Query·Dashboard·Alert Rule·로그 수집 상세와 최종 Evidence는 12 또는 Delivery / Observability 역할 문서에서 관리한다.

## 15. 장애 유형별 Recovery

| 증상 | 우선 확인 | 기본 조치 |
| --- | --- | --- |
| Pod 미생성 | Argo Application / Deployment Replica | Git Desired State 확인 |
| `ImagePullBackOff` | Image Digest / Harbor DNS·TLS / Pull 인증 | Artifact/Registry 경로 수정 후 GitOps 재반영 |
| `CreateContainerConfigError` | Secret / ConfigMap / Key | Provider 공급 상태 확인 |
| Startup 실패 | Process / Env / Config | Pod Event·Log 확인 후 App 또는 Config 수정 |
| Readiness 실패 | Provider readiness / DB / Redis | Provider별 분리 확인 |
| DB TLS 실패 | CA / Endpoint / Secret / MaxScale | CA·TLS 계약 확인 |
| Redis 실패 | Service DNS / Redis Runtime | Redis Service·Pod·연결 확인 |
| Migration 실패 | Job / DB / Replication | Runtime 활성화 중단, 새 승인 후 새 Job |
| Rollout 실패 | 신규 Revision | Git Revert 또는 수정 PR |
| Route 실패 | HTTPRoute Condition / Service | Route Desired State 수정 또는 Revert |
| 2 Replica 전용 장애 | Shared State / Runner / Realtime | 1 Replica로 GitOps 복귀 후 원인 분석 |

## 16. 재실행 / 멱등성 기준

### 16.1 GitOps 변경

동일 Desired State를 다시 Merge하는 방식이 아니라 현재 `main`과 필요한 변경분을 기준으로 새 PR을 만든다.

### 16.2 Migration

Migration Job은 실패한 기존 Job을 재사용하지 않는다.

```text
실패
→ 원인 확인
→ DB 상태 확인
→ 새 승인
→ 새 Render
→ 새 Job
```

### 16.3 Secret 공급

Secret의 실제 공급 자동화는 `seokpan-infra`의 Ansible + Vault 역할을 사용한다. 값 변경이 필요한 경우 GitOps Manifest에 Secret 값을 직접 기록하지 않는다.

## 17. Rollback Matrix

| 변경 대상 | 기본 Rollback | 주의사항 |
| --- | --- | --- |
| Backend Image | Git Revert → Argo Sync | DB Schema와 호환성 확인 |
| Backend Replica | 이전 Replica Desired State로 Revert | 2→1 복귀 시 Realtime 영향 확인 |
| Frontend Image | Git Revert → Argo Sync | Backend API 계약 호환 확인 |
| HPA | HPA 정책 Revert/제거 | 수동 Scale과 경쟁 금지 |
| HTTPRoute | Route Revert | Gateway Platform 재구축 불필요 여부 확인 |
| ConfigMap | Git Revert 후 Pod 반영 필요성 확인 | `subPath` Mount는 재기동 필요 가능 |
| Migration | DB 상태 기반 Restore/보정 판단 | 단순 Git Revert로 Schema 복구 가정 금지 |
| Secret | Infra Ansible/Vault로 이전 값 복구 | Secret 값을 Git에 저장하지 않음 |

## 18. 12 검증·측정 계획으로의 인계

11은 실행 방법을 제공하지만 다음은 12의 책임이다.

```text
정식 Test Case ID
부하 조건
CPU/Memory Target 근거
HPA Trigger 기준
응답시간 / 처리량
Failover / Recovery 시간
RTO / RPO
PASS / FAIL 판정
Metric Query
Log / Screenshot / Run ID
최종 Evidence Link
```

11에서 실행한 절차는 12에서 재현 가능한 Test Case와 Evidence로 연결되어야 한다.

## 19. 완료 기준

본 Runbook 자체의 완료 기준은 다음과 같다.

- 실제 Repository 자산과 실행 절차가 일치한다.
- 현재 없는 자산을 존재한다고 표현하지 않는다.
- Migration의 승인형 One-shot 경계를 보존한다.
- Backend 1 Replica → 2 Replica → HPA 순서를 유지한다.
- Frontend → HTTPRoute → HTTPS/WSS 순서가 명확하다.
- GitOps 변경과 Cluster 직접 변경의 경계가 명확하다.
- 실패 시 중단·복구·Rollback 경로가 있다.
- 09와 책임 중복이 없고 12의 Test/Measurement/Evidence를 침범하지 않는다.
- Current-State README의 stale 내용은 별도 Issue/PR로 추적한다.

서비스 구현이 진행되면서 실제 Manifest·명령·파일 경로가 확정되면 본 Runbook의 Planned 절차를 실제 자산 기준으로 현행화한다.
