# 石나가는 판단 — 1차 프로젝트 Current State

- 기준일: **2026-09-23**
- 목적: 1차 프로젝트 종료 시점의 Actual State, Validation Backlog, 미구현·Deferred, Known Limitation, 2차 재평가 대상을 한 문서에서 추적한다.
- 성격: 01~08 기획·설계 Baseline을 대체하지 않는다. **현재 실제 상태를 연결하는 Closeout Snapshot**이다.

## 1. 기준점

이 문서의 작성 기준 Repository `main`은 다음과 같다.

| Repository | 기준 SHA | 역할 |
| --- | --- | --- |
| `seokpan-app` | `55b1c6185c3fc235bd31186e4e3a9d689786a1ca` | Application Source, CI 정의, Application 내부 문서 |
| `seokpan-gitops` | `cdeb05c72fe92563d39bc07061b8ea6d9263ab53` | Kubernetes Desired State, Argo CD, Platform/CICD/Observability Manifest |
| `seokpan-infra` | `4d382358fbfa6fd512993442a891d19c229e01f0` | On-prem Infra, Network, DB/Storage, Secret 공급, Ansible 자동화 |
| `seokpan-docs` | PR #156 branch가 `main` `1039bfa3722d591d26917ff81879412fb4ca3695`까지 재대조됨; 최종 상태는 본 문서가 포함된 Git revision | 공용 설계·변경·Runbook·Validation·Troubleshooting |

`seokpan-app/main`의 최신 Commit은 README 정리 Commit을 포함한다. 최종 Application 제품 변경은 Room 공통 2열 Layout까지 반영된 Source Freeze 계열이며, 이후 Jenkins Image Pipeline과 GitOps Promotion으로 실제 Desired State가 갱신됐다.

상태 표현은 다음과 같이 사용한다.

- **IMPLEMENTED + VALIDATED** — 구현과 필요한 핵심 Runtime/Evidence가 확인됨
- **IMPLEMENTED / VALIDATION PENDING** — Source/Desired State는 완료됐지만 추가 강화 검증이 남음
- **NOT IMPLEMENTED** — 1차에서 구현하지 않음
- **DEFERRED** — 1차 완료조건에서 제외하고 후속 재평가
- **KNOWN LIMITATION** — 현재 알고 있는 보장 범위 또는 제약
- **HANDOFF / RE-EVALUATE** — 2차에서 요구사항·환경 기준으로 다시 판단

## 2. 문서 Source of Truth 관계

```text
01~08
= 확정 당시 기획·설계 Baseline

PROJECT_CHANGES.md
= Baseline 이후 왜 결정이 변경됐는지

CURRENT_STATE.md
= 종료 시점에 실제 어디까지 구현·검증됐는지

09
= 역할별 실행·통합 실시설계

10
= GitHub 협업·Repository 운영

11
= 재실행 가능한 구축·운영 Runbook

12
= Test/Measurement/Evidence 계약과 실제 결과

Troubleshooting
= 실제 장애 → Root Cause → 조치 → 검증

Implementation Repository
= 현재 코드·Manifest·자동화의 최종 Source of Truth
```

과거 Baseline의 계획값을 현재 Runtime 결과처럼 사용하지 않는다.

## 3. 현재 실제 구조

### 3.1 On-prem / Kubernetes

현재 1차 MVP의 핵심 Platform은 다음 구조로 구현됐다.

- Kubernetes Control Plane 3대
- Worker 2대
- kubeadm
- Calico VXLAN
- Argo CD App-of-Apps
- Gateway API 기반 Application Route
- Backend / Frontend Kubernetes Workload
- Redis StatefulSet + PVC + AOF
- MariaDB Primary/Replica + MaxScale
- NFS
- Jenkins
- Harbor
- Prometheus / Grafana / Loki / Grafana Alloy / Alertmanager

원 목표 18 VM에서 MVP는 `lb-02`, `maxscale-02`를 제외한 축소 구조를 사용한다. 이 축소는 역할·인터페이스를 삭제한 것이 아니라 1차 기간에서 HA Instance 수를 줄인 것이다.

### 3.2 Application State Ownership

현재 Application은 다음 책임 경계를 사용한다.

| 상태 | 권위 저장소 / 책임 |
| --- | --- |
| Member / Game Result / Move / Rating 등 영속 상태 | MariaDB |
| Session / Room / Ready / Connection Generation / Game·Turn Runtime / Vote | Redis |
| Application Source / Domain Contract | `seokpan-app` |
| Kubernetes Desired State | `seokpan-gitops` |
| DB/Secret/CA/Infra 공급 | `seokpan-infra` |

MariaDB와 Redis의 역할을 서로 대체하지 않는다. Redis Runtime 유실 시 MariaDB Authority로 복구 가능한 범위와 실제 Runtime 일관성은 별도 Recovery Evidence로 관리한다.

## 4. 구현·검증 완료

### 4.1 Application Runtime Integration — IMPLEMENTED + VALIDATED

A-01~A-10 Application Roadmap의 Runtime Integration은 완료됐다.

확인된 범위:

- MariaDB / Redis Production Provider 조립
- 승인형 One-shot Alembic Migration 경로
- Backend 1 Replica Provider Gate
- Backend 2 Replica Scale-out 및 Worker 분산
- Frontend 2 Replica
- Redis Session 공유 기본 경로
- MaxScale TLS 실제 Backend/Alembic 연결
- Gateway HTTPS / API / WebSocket
- Let's Encrypt Production Certificate
- Windows Host / Linux VM 외부 접속
- Core Game Gateway Browser Happy Path
- Rankings / Member Statistics
- Room Chat cross-Pod 권한 문제 수정 및 Browser Gate

주요 근거:

- `seokpan-app#3`
- `seokpan-app#22`
- `seokpan-app#50`
- `seokpan-app#74`
- `seokpan-app#94`
- `seokpan-gitops#7`
- `seokpan-infra#188`

### 4.2 Final Source / UX — IMPLEMENTED + VALIDATED

최종 Application Source Freeze 이후:

- Frontend Unit **267/267 PASS**
- Browser UI **18/18 PASS**
- Full Browser E2E **1/1 PASS**
- Desktop / Tablet / Mobile 시각 감사
- Room viewport / Board / Chat 시연 Hotfix
- WAITING / PLAYING 공통 2열 Layout
- 내부 Participant ID neutral fallback
- Global Shell / Navigation 역할 분리
- Local Operation Feedback Source 및 기본 UI 회귀

후속 Frontend 변경은 Jenkins Image Pipeline과 GitOps Promotion PR을 통해 최종 Desired State에 반영됐다. Realtime failure-path의 강화 Browser/Provider 검증은 별도 #112에서 관리한다.

### 4.3 CI/CD / GitOps — IMPLEMENTED + VALIDATED

현재 정상 Application Delivery 흐름:

```text
App main
→ Jenkins 검증 / Build / Scan / Health
→ Harbor Final Digest
→ Component Impact 판정
→ GitOps Promotion Branch / Commit / PR 자동 생성
→ 사람 Review / 1 Approval
→ Merge
→ Argo CD Sync
→ Kubernetes
```

검증된 경계:

- GitOps main 직접 push 없음
- auto-merge 없음
- 사람 승인 유지
- Component별 Source SHA provenance
- 비영향 Component 불필요 rollout 방지
- 실제 Promotion PR #132/#133/#134 생성
- Jenkins BUILD_URL 실제 기록
- Argo CD Self-Heal PASS
- Git Revert 기반 Runtime Rollback PASS

### 4.4 Observability — IMPLEMENTED + VALIDATED

Application Metrics:

- Backend `/metrics`
- ServiceMonitor
- Backend 2 Replica Prometheus Target `UP`
- Application Metric Query PASS

Application Logging:

```text
Backend structured stdout
→ containerd Pod log
→ Grafana Alloy
→ Loki
```

실제 Runtime Harness에서 structured error event와 Context를 확인하고 Loki에서 동일 event를 조회했다.

근거:

- `seokpan-app#90`
- `seokpan-gitops#91`
- `seokpan-gitops#109`

### 4.5 Data / Recovery Evidence

팀 공용 Evidence 기준으로 다음 정량·복구 결과가 존재한다.

- MariaDB DR-01
  - Isolated RTO: 27.376초
  - Production 단일 노드 서비스 재개 RTO: 1분 29초
  - 양쪽 이중화 완전 정상화 RTO: 4분 0초
  - 실측 RPO: 3건
- etcd DR-02
  - 3-member Restore 및 Kubernetes Object 검증
  - RTO: 52초
- Redis DR-03
  - AOF/PVC 손상 상황의 1차 Infra 레벨 복구 검증
  - MariaDB Move Authority를 기준으로 Game State 재구성 가능성 확인

각 역할의 상세 결과와 보장 범위는 해당 12 문서 및 원 Issue를 사용한다.

## 5. 구현 완료 / 강화 Validation Pending

### 5.1 Realtime·2 Replica·Reconnect

현재 Source에는 다음 계약이 구현되어 있다.

- 10초 reconnect grace
- explicit Leave/Logout 즉시 처리
- disconnected Ready의 Start 자격 제외
- shared `connection_generation` / `connected`
- connection replacement / reconnect-required
- late disconnect 보호
- Safe Leave
- uncertain mutation 자동 replay 금지
- Realtime 상태 사용자 피드백

다음은 **구현 완료 여부와 분리한 강화 Validation**이다.

Canonical: `seokpan-app#112`

- F07 실제 cross-Pod WebSocket replacement
- F08 실제 Redis connected/generation 수렴
- setup/provider failure 경계
- 10초 grace reconnect / expiry handoff
- blocked/disconnected Safe Leave
- BFCache 정상 lifecycle과 실제 복귀 실패 구분
- 실제 failure-path Realtime presentation

이 항목을 미검증이라는 이유로 이미 구현된 Source를 미구현으로 되돌려 기록하지 않는다.

### 5.2 P4 Stability / Game Lifecycle

2026-09-23 Closeout에서 `seokpan-app#76/#86/#88/#85`의 구현 상태와 강화 Validation을 분리했다.

현재 판정:

- `#76` — F01~F16을 Complete / Source Complete / NOT REQUIRED / Deferred / Validation Pending으로 재분류 완료
- `#86` — Game lifecycle Source 구현 완료, Issue completed
- `#88` — Realtime Source/정책 구현 완료, App 내부 정책 문서 정합화 PR #113 이후 구현 Issue 종료 예정
- `#85` — Local Operation / Realtime Presentation 구현 완료, Issue completed
- 실제 Provider·2-Pod·Failure Boundary·Measurement는 `seokpan-app#112`가 Canonical Validation Backlog

특히 Game lifecycle은 다음 경계를 유지한다.

```text
Captured lifecycle Source = IMPLEMENTED / main 반영
Production lifecycle mode = legacy
Captured production activation = NOT PERFORMED / DEFERRED
```

현재 `seokpan-gitops/apps/backend/configmap.yaml`에는 `SEOKPAN_GAME_LIFECYCLE_MODE`가 없고 Application 기본값은 `legacy`다. 따라서 captured lifecycle이 Production에서 활성화됐다고 주장하지 않는다.

`#112`에서 추가로 추적하는 범위:

- V-01~V-04 — Realtime / Safe Leave / Reconnect / failure-path UX
- V-05 — Room admission / Session concurrency
- V-06 — Game lifecycle / Recovery / captured 전환 검증
- V-07 — F10/F13 Measurement

이 Validation이 남았다는 이유로 final main에 이미 통합된 Source를 미구현으로 되돌려 기록하지 않으며, 실제 미실행 항목을 PASS로 기록하지도 않는다.

## 6. 1차에서 구현하지 않은 항목

### HPA — NOT IMPLEMENTED / DEFERRED

Backend 2 Replica Scale-out은 검증했지만 HPA는 구현하지 않았다.

미확정 상태에서 임의로 정하지 않은 항목:

- Resource Metrics 경로
- CPU / Memory Request·Limit 근거
- Scale Target / Threshold
- min/max Replica
- Argo CD와 HPA의 `spec.replicas` 소유권

따라서 HPA 결과를 1차 성과로 주장하지 않는다.

### ANALYSIS Runtime — NOT IMPLEMENTED

1차 MVP에는 실제 AI 분석 Runtime, 모델/API, Redis Streams 분석 Pipeline, DLQ를 구현하지 않았다.

Frontend의 AI 관련 표현이 존재하더라도 실제 Runtime 분석 기능이 구현된 것으로 해석하지 않는다.

## 7. Deferred / Known Limitation

### 7.1 Deferred

- Backend HPA
- Captured Game lifecycle Production 활성화
- 추가 `lb-02` / `maxscale-02` HA
- Redis Sentinel / Redis Cluster
- ANALYSIS Runtime
- MaxScale Exporter 실구현
- 일부 추가 Alert Rule
- 2차 Hybrid Cloud / OpenShift / ROSA / Terraform 상세 설계

### 7.2 Known Limitation / Validation Boundary

- Production Game lifecycle은 현재 `legacy`; captured Source는 main에 있으나 운영 전환은 Deferred
- Realtime reconnect / replacement의 일부 failure boundary는 #112의 강화 검증 대상
- 전체 P4 Performance Baseline은 완료된 수치가 없는 항목을 PASS로 쓰지 않음
- Cross-role DR 이후 Application 정상화는 동일 Run/Revision으로 연결된 Evidence가 확보된 범위만 인정
- 현재 GitOps Promotion은 Repository write/PR 전용 Credential + 사람 승인 구조이며 자동 승인/merge는 하지 않음
- App 내부 Realtime 설명 문서의 10초 정책 Drift 정합화는 App PR #113에서 처리 중

## 8. 2차 프로젝트 인계 — 유지할 계약

환경이 OpenShift / ROSA / Hybrid로 바뀌어도 먼저 보존 여부를 확인해야 할 계약:

- MariaDB 영속 Authority / Redis Runtime State 경계
- Application Domain과 Provider Adapter 책임 분리
- Migration은 일반 Backend Startup/Replica에서 자동 실행하지 않는 승인형 One-shot Gate
- Runtime Credential / Migration Credential 분리
- GitOps Desired State와 실제 Runtime 변경 경계
- 검증 Image → Review 가능한 Desired State 변경 → Runtime이라는 Delivery Evidence Chain
- Secret/CA 값을 Git에 평문 저장하지 않는 경계
- 장애와 사용자 이탈을 혼동하지 않는 Application 정책
- 구현 완료와 Validation PASS를 구분하는 Evidence 원칙

## 9. 2차에서 다시 평가할 환경 종속 구현

다음은 1차 구현을 그대로 복사할 대상이 아니라 2차 요구사항·제약조건을 기준으로 다시 비교한다.

- kubeadm ↔ OpenShift / ROSA
- Calico / 현재 Gateway 구현 ↔ 2차 Platform 기본 Networking / Ingress·Gateway
- NFS 기반 Storage ↔ 2차 StorageClass / Managed Storage
- On-prem HAProxy / MaxScale 배치
- HPA / Autoscaling 정책
- Terraform 적용 범위
- Hybrid Network
- Registry / CI/CD Credential 방식
- MaxScale Observability
- DR 대상·RTO/RPO와 Managed Service 책임 경계

## 10. 문서·인계 상태

현재 역할별 09/11/12:

- Kubernetes & Application Integration — 정태훈
- Database / Storage / Recovery — 김상희
- Delivery / Observability — 최유준

Network / External Infra 역할의 별도 09/11/12 문서 세트는 현재 Snapshot에서 확인되지 않는다. 별도 문서가 실제로 필요한지, 또는 Infra Repository·PROJECT_CHANGES·Troubleshooting을 공식 Source로 사용할지는 해당 Owner와 결정한다.

README는 App / Infra / GitOps 모두 2026-09-23 최신 요약형 구조로 merge됐다. Current State와 충돌하는 표현이 발견되면 각 README Owner가 후속 수정한다.

## 11. 주요 Source / Evidence Index

### 공용

- `PROJECT_CHANGES.md`
- `MVP_IMPLEMENTATION_BASELINE.md`
- 09 / 10 / 11 / 12
- `troubleshooting/`

### Application / Platform

- `seokpan-app#3` — Application Roadmap
- `seokpan-app#76` — Stability Parent Closeout / Validation 분리
- `seokpan-app#86` — Game lifecycle Source Closeout / Production legacy 경계
- `seokpan-app#88` — Realtime Source Closeout / Validation #112
- `seokpan-app#82` — UX Parent
- `seokpan-app#90` — Application Logging
- `seokpan-app#94` — Gateway Browser Happy Path
- `seokpan-app#98` — GitOps Promotion Automation
- `seokpan-app#112` — Closeout Realtime Validation Backlog
- `seokpan-gitops#57` — CD / Self-Heal / Rollback
- `seokpan-gitops#109` — F04 Runtime / Loki Evidence

### Closeout / Phase 2

- `seokpan-docs#153` — Current State Closeout
- `seokpan-docs#89` — 1차 → 2차 Handoff 구조 판단
- `seokpan-docs#127` — tjung03 개인 Closeout Final Audit

## 12. Closeout 원칙

1차 종료 시 Open Issue 수를 0으로 만드는 것이 목표가 아니다.

목표는 모든 잔여 상태가 다음 중 하나로 설명 가능하게 수렴하는 것이다.

```text
COMPLETE
VALIDATION PENDING
NOT IMPLEMENTED
DEFERRED
KNOWN LIMITATION
HANDOFF
NO PERSONAL ACTION
```

실제로 수행하지 않은 검증을 완료로 바꾸지 않고, 반대로 강화 검증이 남았다는 이유만으로 이미 구현·통합된 기능을 미구현으로 되돌리지 않는다.
