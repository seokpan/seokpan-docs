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

본 문서는 현재 구현 상태와 Repository 자산을 기준으로 작성한다. 아직 구현되지 않은 HPA, Application HTTPRoute 등은 완료된 실행 절차처럼 표현하지 않고, 구현 전제와 실행 순서를 분리한다.

직접 기준:

- [`09_MVP_실행·통합_실시설계_Kubernetes_Application_Integration_정태훈.md`](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Kubernetes_Application_Integration_정태훈.md)
- [`10_GitHub_협업_및_Repository_운영.md`](../10_GitHub_협업_및_Repository_운영.md)
- [`PROJECT_CHANGES.md`](../PROJECT_CHANGES.md)
- [`MVP_IMPLEMENTATION_BASELINE.md`](../MVP_IMPLEMENTATION_BASELINE.md)
- `seokpan-app` 현재 `main`, Issue, PR
- `seokpan-gitops` 현재 `main`, Issue, PR
- `seokpan-infra` 현재 `main`, Issue, PR
- 실제 Runtime Evidence

현재 상태는 계속 변할 수 있으므로 실행 직전에는 반드시 구현 Repository의 최신 `main`과 열린 Issue/PR을 다시 확인한다.

---

## 2. 실행 원칙

### 2.1 Source of Truth와 Working Directory

실행 명령은 해당 자산을 소유하는 Repository 기준으로 수행한다.

```text
Application Source / CI     → seokpan-app
Kubernetes Desired State    → seokpan-gitops
Infra / Secret Supply       → seokpan-infra
Project Runbook / Evidence  → seokpan-docs
```

특히 아래 상대경로 명령은 `seokpan-gitops` Repository Root에서 실행한다.

```text
apps/backend/**
apps/frontend/**
platform/**
observability/**
argocd/applications/**
```

Runbook에서 `<seokpan-gitops-repo-root>`는 실제 로컬의 `seokpan-gitops` Repository Root를 의미한다.

### 2.2 GitOps 우선

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

운영 상태를 맞추기 위해 `kubectl edit`, 직접 `kubectl apply`, 직접 Scale을 기본 운영 방식으로 사용하지 않는다. 직접 명령은 조회·진단·server-side dry-run·명시적인 검증 절차에 한정한다.

### 2.3 상태 판정

다음을 같은 의미로 취급하지 않는다.

```text
Implemented ≠ Merged ≠ Running ≠ Validated
Image Pushed ≠ GitOps Updated ≠ Argo Synced ≠ Pod Ready
Provider Ready ≠ Consumer Wired ≠ Integration Validated
```

### 2.4 Secret 비노출

Password, 전체 DB URL, Token, Private Key, Secret Value를 콘솔·Issue·PR·문서에 출력하거나 기록하지 않는다.

필요한 경우 Secret의 존재와 Key 이름만 확인한다.

### 2.5 중단 우선

아래 조건에서는 다음 단계로 진행하지 않는다.

- 현재 Release Image Digest 미확정
- 필요한 Secret / ConfigMap 부재
- Active Migration Job 존재
- Migration Gate 실패 또는 후검증 미완료
- Backend 1 Replica First Runtime 실패
- Provider readiness 실패
- GitOps Desired State와 실제 작업 대상 Revision 불일치
- A-10 Production Provider Integration 미완료 상태에서 Runtime 강제 활성화

---

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
| Backend → Redis Consumer | Not Tested |
| DB Runtime/Migration Secret 공급 구조 | Implemented / Validated |
| One-shot Migration 자산 | Implemented / Merged / Static+API Validated |
| 실제 Migration Gate 실행 | Not Tested |
| A-10 Production Provider Integration | In Progress |
| Backend `SEOKPAN_ALLOWED_ORIGINS` JSON 배열 계약 | Runtime 활성화 전 정합화 필요 (`seokpan-gitops#90`) |
| Application HTTPRoute | Not Implemented |
| HPA | Not Implemented |
| Application ServiceMonitor | `.pending`, Runtime Metrics 미검증 |

이 표는 12의 공식 PASS/FAIL Evidence를 대체하지 않는다.

---

## 4. 공통 Pre-check

### 4.1 Repository 상태

작업할 Repository Root에서 다음을 확인한다.

```bash
git status
git branch --show-current
git fetch --prune
git log --oneline --decorate -n 5
```

실제 변경은 별도 Branch에서 수행한다.

### 4.2 Kubernetes Node

```bash
kubectl get nodes -o wide
```

최소 확인:

- Control Plane 3대
- Worker 2대
- 대상 Node `Ready`

### 4.3 Argo CD Application

전체 Application:

```bash
kubectl -n argocd get applications
```

Backend/Frontend/Gateway/Redis의 Target Revision·Sync·Health를 확인한다.

```bash
for app in apps-backend apps-frontend gateway redis; do
  kubectl -n argocd get application "$app" \
    -o custom-columns='NAME:.metadata.name,REVISION:.spec.source.targetRevision,SYNC:.status.sync.status,HEALTH:.status.health.status' \
    --no-headers
done
```

최소 확인:

```text
Application 존재
Target Revision = 의도한 Revision
Sync Status
Health Status
```

### 4.4 Application Namespace

```bash
kubectl -n application get deploy,svc
```

현재 Runtime 활성화 전에는 Backend/Frontend가 `0/0`이어도 정상이다.

### 4.5 Redis

```bash
kubectl -n platform get statefulset,pod,svc,pvc
```

Redis Platform Runtime이 정상인 것과 Backend Consumer 연결 성공은 별도로 판정한다.

### 4.6 Secret / CA 존재와 Key 이름 확인

Secret 값은 출력하지 않는다.

```bash
for secret in backend-db-runtime backend-db-migration; do
  echo "===== ${secret} ====="
  kubectl -n application get secret "$secret" \
    -o go-template='{{range $k,$v := .data}}{{println $k}}{{end}}'
done
```

기대 Key:

```text
backend-db-runtime
├─ SEOKPAN_IDENTITY_DATABASE_URL
└─ SEOKPAN_GAME_DATABASE_URL

backend-db-migration
└─ SEOKPAN_MIGRATION_DATABASE_URL
```

공개 CA ConfigMap은 Key 이름만 확인한다.

```bash
kubectl -n application get configmap seokpan-internal-ca \
  -o go-template='{{range $k,$v := .data}}{{println $k}}{{end}}'
```

기대 Key:

```text
ca.crt
```

Secret 실제 값의 Source of Truth는 `seokpan-infra`의 Ansible + Vault다.

### 4.7 Backend ConfigMap 계약 확인

Backend Runtime 활성화 전 `apps/backend/configmap.yaml`의 Application Settings 계약을 현재 `seokpan-app`과 대조한다.

특히 A-10 기준 `SEOKPAN_ALLOWED_ORIGINS`는 JSON 배열 표현이 필요하다.

목표 예:

```yaml
SEOKPAN_ALLOWED_ORIGINS: '["https://game.seokpan.soldesk.store"]'
```

현재 정합화 작업은 `seokpan-gitops#90`에서 추적한다.

---

## 5. Application Artifact 준비

### 5.1 Release Source

A-09에서 Image Pipeline Capability가 검증되었더라도 A-10 Source가 변경되면 현재 Runtime이 사용할 Image는 새 `main` 기준으로 다시 생성·검증해야 한다.

```text
A-10 Source Merge
→ Jenkins main Pipeline
→ Test / Build
→ Harbor Push
→ Scan / Acceptance
→ Digest 확인
→ Runtime Candidate 확정
```

과거 A-09 Artifact를 A-10 최종 Artifact로 재사용하지 않는다.

### 5.2 Image 정책

실제 Runtime 활성화에는 `latest` 또는 `git-pending`을 사용하지 않는다.

목표 형식:

```text
harbor.seokpan.soldesk.store/seokpan/backend@sha256:<verified-digest>
harbor.seokpan.soldesk.store/seokpan/frontend@sha256:<verified-digest>
```

Digest는 실행 시점의 검증된 Harbor Artifact에서 가져온다.

Kustomize Image 계약은 실제 `kubectl kustomize` 렌더 결과로 다시 확인한다.

---

## 6. DB / CA / Provider Pre-check

### 6.1 DB Endpoint

공식 DB Endpoint:

```text
db.seokpan.soldesk.store:3306
```

Runtime Credential과 Migration Credential을 혼용하지 않는다.

```text
Runtime   → identity_svc / game_svc
Migration → db_admin
```

### 6.2 CA

공개 CA 계약:

```text
ConfigMap: seokpan-internal-ca
Key:       ca.crt
Mount:     /etc/seokpan/pki/ca.crt
Env:       SEOKPAN_DATABASE_CA_FILE=/etc/seokpan/pki/ca.crt
```

CA 파일 존재·지문·서버 인증서 검증과 실제 Backend→MaxScale TLS 연결 성공은 서로 다른 결과로 본다.

### 6.3 Migration Gate 선행조건

Mutation Migration 전에 최소 다음이 충족되어야 한다.

- 승인된 Backend Image Digest
- `backend-db-migration` Secret 준비
- DB 상태 확인
- Replication 상태 확인
- Backup / Restore 가능 상태 확인
- CA 준비
- Active Migration Job 없음
- 실행 승인 Reference 존재

DB·Replication·Backup/Restore의 최종 준비 판정은 Data / Storage / Recovery 담당 근거와 연결한다.

---

## 7. One-shot Migration Runbook

Migration 자산은 `seokpan-gitops/apps/backend/migration/`에 있으며 일반 Backend Kustomization과 분리되어 있다. Argo CD Auto-Sync로 DDL을 실행하지 않는다.

### 7.1 실행 위치

```bash
cd <seokpan-gitops-repo-root>

test -f apps/backend/migration/render-job.py
test -f apps/backend/migration/job-template.yaml
```

### 7.2 Active Job Guard

```bash
kubectl get jobs -n application \
  -l app.kubernetes.io/name=backend-db-migration \
  -o custom-columns='NAME:.metadata.name,ACTIVE:.status.active,SUCCEEDED:.status.succeeded,FAILED:.status.failed'
```

`ACTIVE` 값이 있는 Job이 존재하면 새 Mutation Job을 생성하지 않는다.

### 7.3 DB Audit와 Action 선택

Migration Action을 항상 `upgrade-head`로 가정하지 않는다.

```text
current        → Read-only Revision 확인
stamp-baseline → 기존 Baseline Schema를 승인된 Revision으로 Stamp
upgrade-head   → 승인된 Migration을 Head까지 적용
```

어떤 Action을 사용할지는 App #22의 DB Audit, 승인, 현재 Revision 결과를 기준으로 결정한다.

### 7.4 Read-only current Render

```bash
python3 apps/backend/migration/render-job.py \
  current \
  --image 'harbor.seokpan.soldesk.store/seokpan/backend@sha256:<verified-digest>' \
  --output /tmp/backend-migration-current.yaml
```

### 7.5 Mutation Render

예:

```bash
python3 apps/backend/migration/render-job.py \
  upgrade-head \
  --image 'harbor.seokpan.soldesk.store/seokpan/backend@sha256:<verified-digest>' \
  --approval-ref 'seokpan/<repo>#<issue-number>:issuecomment-<comment-id>' \
  --output /tmp/backend-migration-upgrade.yaml
```

Approval Reference에는 Credential·DB URL·Secret Value를 기록하지 않는다.

### 7.6 Kubernetes API Dry-run

생성된 실행 파일을 변수로 지정한다.

```bash
RUN_YAML=/tmp/backend-migration-upgrade.yaml
kubectl create --dry-run=server -f "$RUN_YAML"
```

`current` Action을 실행하는 경우 `RUN_YAML`을 해당 Read-only Render 파일로 지정한다.

Dry-run 성공은 실제 DB Migration 성공을 의미하지 않는다.

### 7.7 실제 실행

모든 Gate가 충족된 경우에만 새 Job을 생성한다.

```bash
kubectl create -f "$RUN_YAML"
```

완료·실패한 기존 Job에 `kubectl apply`하여 재사용하지 않는다.

실행 후 생성된 Job 이름을 다시 확인한다.

```bash
kubectl get jobs -n application \
  -l app.kubernetes.io/name=backend-db-migration
```

필요한 경우 해당 Job의 Pod/Log를 조회하되 Credential·전체 URL이 노출되지 않는지 주의한다.

### 7.8 실행 후 확인

최소 확인 대상:

```text
Job terminal status
Alembic current
Replication
MaxScale Read/Write
기존 데이터 보존
```

Backend Runtime으로 넘어가기 위한 조건은 **승인된 Migration Gate 수행과 후검증 완료**다. 실제 Mutation이 필요한지 여부는 DB Audit 결과를 따른다.

구체 PASS/FAIL 기준과 Evidence 형식은 12에서 관리한다.

### 7.9 실패

```text
Migration Failed
→ Backend Runtime 활성화 중단
→ Job / Log 확인
→ DB 상태 확인
→ Replication 확인
→ Rollback / Restore 필요성 판단
→ 원인 수정
→ 새로운 승인
→ 새 Render
→ 새로운 Job 생성
```

Schema 변경은 Git Revert만으로 DB를 원상복구할 수 있다고 가정하지 않는다.

---

## 8. Backend 1 Replica First Runtime

Backend 1 Replica는 실제 Provider Integration의 첫 Runtime Gate다.

### 8.1 선행조건

- A-10 Production Provider Integration 완료
- A-10 변경을 포함한 새 `main` Backend Image Acceptance 완료
- 승인된 Migration Gate 수행 및 후검증 완료
- `backend-db-runtime` Secret 준비
- `seokpan-internal-ca` 준비
- Redis Runtime 준비
- Provider-aware readiness 기준 준비
- `apps-backend` Argo CD Application 존재
- `SEOKPAN_ALLOWED_ORIGINS` JSON 배열 계약 정합화 (`seokpan-gitops#90`)

하나라도 충족되지 않으면 `replicas: 0`을 유지한다.

### 8.2 GitOps 변경 대상

```text
apps/backend/kustomization.yaml
apps/backend/deployment.yaml
apps/backend/configmap.yaml   # Settings 계약 변경이 필요한 경우
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
production Settings 계약 정합
```

Image Digest pinning은 현재 Kustomize 버전의 렌더 결과를 기준으로 적용한다. Kustomize `images` 항목은 image name/tag뿐 아니라 digest 교체도 지원한다.

예:

```yaml
images:
  - name: harbor.seokpan.soldesk.store/seokpan/backend
    newName: harbor.seokpan.soldesk.store/seokpan/backend
    digest: sha256:<verified-digest>
```

### 8.3 Merge 전 Render / API 검증

```bash
cd <seokpan-gitops-repo-root>

kubectl kustomize apps/backend \
  > /tmp/backend-rendered.yaml

kubectl apply --dry-run=server \
  -f /tmp/backend-rendered.yaml
```

Dry-run은 Runtime Provider 연결 성공을 의미하지 않는다.

### 8.4 Merge 후 Argo / Rollout 확인

```bash
kubectl -n argocd get application apps-backend

kubectl -n application get deployment backend

kubectl -n application get pods \
  -l app.kubernetes.io/name=backend -o wide

kubectl -n application rollout status \
  deployment/backend
```

### 8.5 Probe와 Provider

Deployment 기준 Health Endpoint:

```text
Startup   /health/startup
Liveness  /health/live
Readiness /health/ready
```

Probe 성공과 DB/Redis Provider E2E 성공을 동일하게 보지 않는다.

Backend Pod 기동 후 최소 연결 범위:

```text
Backend → MaxScale TLS
Backend → Identity DB
Backend → Game DB
Backend → Redis Service DNS
Provider-aware Readiness
```

구체 Test Case와 Evidence는 12에서 관리한다.

### 8.6 실패 시

신규 Image 또는 Runtime 변경으로 Backend가 정상화되지 않으면 Cluster를 직접 수정하지 않고 Git Desired State를 되돌린다.

```text
문제 Revision 식별
→ Git Revert 또는 수정 PR
→ Review / Merge
→ Argo CD Sync
→ 이전 Desired State 회복 확인
```

첫 Runtime 이전 상태로 복귀해야 하는 경우 **pre-activation Git Revision 전체**를 기준으로 복구한다. `replicas`와 Image만 임의 조합해 새로운 미검증 상태를 만들지 않는다.

---

## 9. Backend 2 Replica Scale-out

1 Replica 통합 검증 전에는 2 Replica로 확장하지 않는다.

### 9.1 선행조건

- Backend 1 Replica Running
- DB/Redis Provider 연결 정상
- Migration Gate 후검증 완료
- Realtime/Runner가 Process-local Authority에 의존하지 않는 A-10 상태

### 9.2 GitOps Scale-out

`apps/backend/deployment.yaml`의 Desired Replica를 2로 변경하고 PR/Merge한다.

Merge 전:

```bash
cd <seokpan-gitops-repo-root>
kubectl kustomize apps/backend > /tmp/backend-rendered.yaml
kubectl apply --dry-run=server -f /tmp/backend-rendered.yaml
```

Merge 후:

```bash
kubectl -n application get deployment backend
kubectl -n application get pods \
  -l app.kubernetes.io/name=backend -o wide
kubectl -n application rollout status deployment/backend
```

### 9.3 확인 범위

```text
Pod 2개 Ready
Worker 배치 상태
Redis 공유 Runtime State
Redis Pub/Sub 기반 변경 알림
Runner 중복 실행 방지
Reconnect 수렴
Cross-replica Realtime
```

Worker 분산 여부 자체를 성공 기준으로 임의 확정하지 않는다. 정식 배치·장애·동시성 PASS/FAIL은 12에서 정의한다.

### 9.4 실패 시 축소

2 Replica에서만 문제가 발생하면 원인 분석 동안 GitOps를 통해 검증된 1 Replica Revision으로 복귀한다.

```text
2 Replica 문제
→ 1 Replica 정상 Revision 식별
→ Git Revert / 수정 PR
→ Merge
→ Argo CD Sync
→ 1 Replica 정상 상태 확인
```

---

## 10. HPA / Workload Distribution

현재 Backend HPA와 Metrics Server 연계 Application 자산은 구현 완료 상태로 확인되지 않는다. 아래는 **Planned Runbook**이다.

09에서 정한 순서를 유지하여, **Backend 2 Replica 수동 Scale-out 경로가 먼저 성립한 뒤 HPA로 넘어간다.**

### 10.1 선행조건

- Backend 1 Replica 통합 성공
- Backend 2 Replica 수동 Scale-out 경로 검증 완료
- Container Resource Request / Limit 결정
- Metrics Server 또는 Resource Metrics 경로 준비
- HPA와 Argo CD 사이의 `replicas` ownership 계약 확정

### 10.2 HPA / Argo CD Replica Ownership

HPA는 `Deployment.spec.replicas`를 변경한다. 반면 Argo CD `selfHeal`이 정적 `replicas` 값을 계속 소유하면 HPA와 Argo CD가 같은 필드를 두고 경쟁할 수 있다.

HPA를 Merge하기 전에 프로젝트는 아래 방식 중 하나를 명시적으로 선택하고 실제 Diff/Sync 동작을 검증해야 한다.

```text
A. HPA 활성 구간에서는 Deployment Desired State에서 정적 replicas 소유를 제거하고 실제 Render/Apply 동작을 검증
또는
B. apps-backend Application에 /spec/replicas ignoreDifferences를 정의하고,
   Sync 단계에서도 해당 예외를 존중하도록 RespectIgnoreDifferences=true를 함께 적용
```

방식 B 개념 예:

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
  syncPolicy:
    syncOptions:
      - RespectIgnoreDifferences=true
```

`ignoreDifferences`만 설정하면 기본적으로 Diff 계산과 Sync 적용이 동일하게 처리된다고 가정하지 않는다. 현재 Argo CD 구조와 live Deployment 상태에서 실제 동작을 검증한 뒤 확정한다.

### 10.3 구현 순서

```text
Resource Request / Limit 정의
→ Resource Metrics 경로 준비
→ Replica Ownership 계약 확정
→ HPA Desired State 작성
→ Kustomize / Argo Application 정합화
→ 정적/API 검증
→ GitOps PR
→ Review / Merge
→ Argo CD Sync
→ HPA 상태 확인
```

CPU/Memory Target, `minReplicas`, `maxReplicas`는 근거 없이 본 문서에서 임의 확정하지 않는다.

### 10.4 Runtime 확인 예

Metrics API 준비 후에만 실행한다.

```bash
kubectl -n application get hpa
kubectl -n application describe hpa backend
kubectl top pods -n application
kubectl top nodes
```

### 10.5 Rollback

HPA 정책이 문제를 만들면:

```text
HPA Desired State Revert/제거
→ HPA용 Replica Ownership 예외도 함께 원복
→ 검증된 Static Replica Desired State 복구
→ Argo CD Sync
```

수동 Scale과 HPA가 동시에 Desired State를 경쟁하도록 운영하지 않는다.

---

## 11. Frontend Runtime

### 11.1 현재 상태

```text
replicas: 0
image: git-pending
apps-frontend Child Application: Root 편입 완료
```

GitOps 관리 편입과 실제 Frontend Runtime 활성화는 별개다.

### 11.2 선행조건

- 실제 Frontend Image Digest 확보
- Container Runtime Smoke 확인
- `apps-frontend` Argo CD Application 확인
- Backend Runtime과 동일 Origin 계약 준비

Frontend는 환경별 Backend 절대 URL을 Image에 굽지 않고 같은 Origin의 `/api/v1`, `/ws/v1`을 사용한다.

### 11.3 GitOps 변경 대상

```text
apps/frontend/kustomization.yaml
apps/frontend/deployment.yaml
```

실제 초기 Replica 수는 활성화 PR에서 근거를 가지고 확정한다.

### 11.4 Merge 전 검증

```bash
cd <seokpan-gitops-repo-root>

kubectl kustomize apps/frontend \
  > /tmp/frontend-rendered.yaml

kubectl apply --dry-run=server \
  -f /tmp/frontend-rendered.yaml
```

### 11.5 Merge 후 확인

```bash
kubectl -n argocd get application apps-frontend

kubectl -n application get deployment frontend
kubectl -n application get service frontend

kubectl -n application get pods \
  -l app.kubernetes.io/name=frontend -o wide

kubectl -n application rollout status \
  deployment/frontend
```

Frontend Runtime 활성화 성공과 Browser E2E 성공은 별도로 판정한다.

---

## 12. Application HTTPRoute / Gateway Integration

현재 Gateway Platform은 구성되어 있으나 Application HTTPRoute는 구현 완료 상태로 확인되지 않는다. 아래는 **Planned Runbook**이다.

### 12.1 목표 Route 계약

```text
/       → frontend:8080
/api/v1 → backend:8000
/ws/v1  → backend:8000
```

### 12.2 선행조건

- Frontend Service Ready
- Backend Service Ready
- Gateway Platform Synced/Healthy
- TLS / Hostname Platform 경로 정상

### 12.3 구현 순서

```text
Application HTTPRoute Manifest 작성
→ Backend/Frontend Service 참조 확인
→ Hostname / Path Rule 확인
→ Kustomize 포함
→ server-side dry-run
→ GitOps PR
→ Review / Merge
→ Argo CD Sync
→ HTTPRoute Conditions 확인
→ HTTPS / WSS Runtime 확인
```

아직 Route 자산 경로와 이름이 확정되지 않았으므로 본 문서에서 임의 파일명을 만들지 않는다.

### 12.4 적용 후 확인 예

```bash
kubectl get gateway -A
kubectl get httproute -A
kubectl describe httproute -n application <actual-route-name>
```

실제 Route 이름은 구현된 Manifest를 기준으로 사용한다.

### 12.5 실패 시

Route 변경으로 외부 경로가 깨지면 Git Revert로 이전 Route Desired State를 복구한다. Gateway Platform 자체가 정상인 경우 GatewayClass/Gateway를 불필요하게 재설치하지 않는다.

---

## 13. Argo CD 운영

### 13.1 Desired / Live 확인

```bash
kubectl -n argocd get applications
```

필요한 Application은 Target Revision·Sync·Health를 구체적으로 조회한다.

### 13.2 Self-Heal

`apps-backend`, `apps-frontend`는 automated sync와 `selfHeal: true` 경로에 편입되어 있다.

Live Resource 직접 변경을 장기 운영 상태로 만들지 않는다. Self-Heal 동작 자체를 시험하는 절차와 Evidence는 12에서 관리한다.

### 13.3 Rollback

기본 GitOps Rollback:

```text
정상 Revision 식별
→ Git Revert
→ PR
→ Review
→ Merge
→ Argo CD Sync
→ Kubernetes 복구 확인
```

Argo CD UI에서 임의 Revision을 장기 Source of Truth로 만들지 않고 Git `main`과 Desired State를 다시 일치시킨다.

---

## 14. Application Observability 연결

Observability Platform 자체의 구축·운영은 Delivery / Observability 담당 영역이다. Kubernetes & Application Integration은 Application Runtime이 관측 대상이 될 수 있도록 Application 계약을 제공한다.

### 14.1 Metrics 현재 자산

현재 GitOps에는 Application용 pending 자산이 존재한다.

```text
observability/servicemonitor-app.yaml.pending
```

`.pending` 상태에서는 정식 Argo CD Desired State에 편입하지 않는다.

Current-State 정합화는 `seokpan-gitops#86`에서 추적한다.

### 14.2 Metrics 활성화 전 실제 대조

문서에 특정 과거 selector 값을 고정해서 신뢰하지 않고 실행 직전 실제 Backend Service와 pending ServiceMonitor를 대조한다.

Backend Service selector 확인:

```bash
kubectl -n application get service backend \
  -o jsonpath='{.spec.selector}{"\n"}'
```

Repository에서도 다음을 함께 확인한다.

```text
apps/backend/service.yaml
observability/servicemonitor-app.yaml.pending
```

최소 조건:

- Backend Runtime Running
- Backend가 실제 `/metrics`를 제공
- ServiceMonitor selector가 실제 Backend Service label과 일치
- Service Port `http`와 ServiceMonitor endpoint 일치
- Delivery / Observability 담당 Review

### 14.3 Metrics 활성화 순서

```text
Backend Runtime Running
→ Backend /metrics 제공 확인
→ Backend Service Label / Port 확인
→ pending ServiceMonitor 정합화
→ .pending 제거 / 정식 YAML 편입
→ GitOps PR
→ Delivery / Observability Review
→ Merge
→ Argo CD Sync
→ Prometheus Target 확인
```

Observability Platform Running만으로 Application Metrics Integration 완료를 선언하지 않는다.

### 14.4 Application Log 연결

현재 Alloy는 Kubernetes Pod를 discovery하고 다음 라벨을 Loki 전달 경로에 부여하도록 구성되어 있다.

```text
namespace
pod
container
node_name
```

Backend/Frontend Runtime 활성화 후 최소 확인 범위:

```text
application Namespace Pod가 Alloy discovery 대상에 포함됨
Container stdout/stderr 로그가 Loki 경로로 수집됨
namespace / pod / container / node_name 라벨이 실제 Pod와 일치
Backend의 SEOKPAN_INSTANCE_ID와 Pod Identity를 혼동하지 않음
Credential / 전체 DB URL / Secret 값이 Application Log에 노출되지 않음
```

Alloy/Loki Platform Running과 실제 Application Log 수집 성공은 별도로 판정한다.

Metric Query·Log Query·Dashboard·Alert Rule·최종 Evidence는 12 또는 Delivery / Observability 역할 문서에서 관리한다.

---

## 15. 장애 유형별 Recovery

| 증상 | 우선 확인 | 기본 조치 |
| --- | --- | --- |
| Pod 미생성 | Argo Application / Deployment Replica | Git Desired State 확인 |
| `ImagePullBackOff` | Image Digest / Harbor DNS·TLS / Pull 인증 | Artifact/Registry 경로 수정 후 GitOps 재반영 |
| `CreateContainerConfigError` | Secret / ConfigMap / Key | Provider 공급 상태 확인 |
| Startup 실패 | Process / Env / Config | Pod Event·Log 확인 후 App 또는 Config 수정 |
| Readiness 실패 | Provider readiness / DB / Redis | Provider별 상태 분리 확인 |
| DB TLS 실패 | CA / Endpoint / Secret / MaxScale | CA·TLS 계약 확인 |
| Redis 실패 | Service DNS / Redis Runtime | Redis Service·Pod·연결 확인 |
| Migration 실패 | Job / DB / Replication | Runtime 활성화 중단, 새 승인 후 새 Job |
| Rollout 실패 | 신규 Revision | Git Revert 또는 수정 PR |
| Route 실패 | HTTPRoute Condition / Service | Route Desired State 수정 또는 Revert |
| 2 Replica 전용 장애 | Shared State / Runner / Realtime | 검증된 1 Replica Revision으로 복귀 후 분석 |
| Metrics 미수집 | `/metrics` / Service label / ServiceMonitor / Target | Application·Observability 경계 순차 확인 |
| Logs 미수집 | Pod discovery / Alloy / Loki / labels | Application Pod와 수집 경로 순차 확인 |
| HPA와 Argo 반복 Diff | Replica Ownership 계약 | HPA/Argo 소유권 설정 원복 또는 정합화 |

장애 복구의 공식 측정값과 Evidence는 12에서 관리한다.

---

## 16. 재실행 / 멱등성 기준

### 16.1 GitOps 변경

현재 `main`과 필요한 변경분을 기준으로 새 PR을 만든다. 이미 Merge된 동일 변경을 임의로 반복 적용하지 않는다.

### 16.2 Migration

실패한 기존 Migration Job을 재사용하지 않는다.

```text
실패
→ 원인 확인
→ DB 상태 확인
→ 새 승인
→ 새 Render
→ 새 Job
```

### 16.3 Secret 공급

Secret 실제 공급은 `seokpan-infra`의 Ansible + Vault 경로를 사용한다. GitOps Manifest에 실제 Secret 값을 직접 기록하지 않는다.

Secret을 환경변수로 소비하는 실행 중 Pod는 Secret Object 변경만으로 기존 프로세스 환경변수가 갱신되지 않는다. 실제 적용이 필요한 경우 소비 Pod의 안전한 Rollout 경로를 별도 GitOps 변경과 함께 검토한다.

### 16.4 ConfigMap 공급

`envFrom`으로 읽은 ConfigMap 환경변수도 실행 중 프로세스에 자동 재주입되지 않는다. 또한 `subPath`로 Mount한 CA 파일은 ConfigMap 변경만으로 실행 중 컨테이너의 파일이 갱신되지 않는다.

따라서 ConfigMap/CA 변경 시:

```text
Git Desired State 변경
→ 변경 대상 확인
→ 소비 Pod 재기동 필요성 판단
→ 필요하면 Pod Template 변경을 Git에 반영해 Rollout 유도
→ Argo CD Sync
→ 새 Pod에서 반영 확인
```

직접 `kubectl rollout restart`만 수행하여 Git에 남지 않는 운영 변경을 기본 방식으로 삼지 않는다.

### 16.5 Argo Sync

Argo Sync 문제가 발생했을 때 먼저 Git Revision·Application Source·Diff를 확인한다. Sync 버튼을 반복하여 원인을 덮지 않는다.

---

## 17. Rollback Matrix

| 변경 대상 | 기본 Rollback | 주의사항 |
| --- | --- | --- |
| Backend Image / Replica / Config | 정상 Git Revision으로 Revert | DB Schema와 App 호환성 확인 |
| Frontend Image / Replica | 정상 Git Revision으로 Revert | Backend API 계약 호환 확인 |
| HPA | HPA 정책 + Replica Ownership 계약 Revert | Static Replica Desired State도 함께 복구 |
| HTTPRoute | Route Revert | Gateway Platform 재구축 불필요 여부 확인 |
| ConfigMap / CA | Git Revert + 소비 Pod 반영 경로 확인 | `envFrom`, `subPath`는 기존 Pod에 자동 반영되지 않음 |
| Migration | DB 상태 기반 Restore/보정 판단 | 단순 Git Revert로 Schema 복구 가정 금지 |
| Secret | Infra Ansible/Vault로 이전 값 복구 + 소비 Pod 반영 확인 | Secret 값을 Git에 저장하지 않음 |
| ServiceMonitor | Pending/이전 정상 Desired State로 Revert | Application Metrics와 Platform 상태 분리 |

---

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

---

## 19. Runbook 완료 기준

본 Runbook 자체의 완료 기준:

- 실제 Repository 자산과 실행 절차가 일치한다.
- 명령이 문서에서 주장하는 정보를 실제로 확인할 수 있다.
- 실행 위치가 필요한 명령은 Working Directory가 명확하다.
- 현재 없는 자산을 존재한다고 표현하지 않는다.
- Migration의 승인형 One-shot 경계를 보존한다.
- Backend 1 Replica → 2 Replica → HPA 순서를 유지한다.
- HPA와 Argo CD의 Replica Ownership 충돌을 사전에 방지한다.
- Frontend → HTTPRoute → HTTPS/WSS 순서가 명확하다.
- GitOps 변경과 Cluster 직접 변경의 경계가 명확하다.
- ConfigMap/Secret 변경 시 실행 중 Pod 반영 경계를 명확히 한다.
- Application Metrics와 Log 연결 경계를 모두 다룬다.
- 실패 시 중단·복구·Rollback 경로가 있다.
- 09와 책임 중복이 없고 12의 Test/Measurement/Evidence를 침범하지 않는다.
- Current-State stale 내용은 구현 Repository Issue/PR로 분리 추적한다.

서비스 구현이 진행되면서 실제 Manifest·명령·파일 경로가 확정되면 Planned 절차를 실제 자산 기준으로 현행화한다.
