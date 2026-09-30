# 石나가는 판단 — 1차 프로젝트 Current State

- 기준일: **2026-09-23** (1차 종료 스냅샷)
- 후속 현행화: **2026-09-27** (GitHub 기록·Source 기준, 운영 서버 재조회 없음)
- 문서 인덱스 현행화: **2026-09-29 KST** (Network 문서 추가·참조 범위 반영. Runtime 및 App #112의 진행 중 검증 결과는 이번 변경에서 재판정하지 않음)
- Closeout 현행화: **2026-09-29 KST** (App PR #119/#120, Infra PR #220, GitOps PR #138 merge와 Docs #89 최종 A안 반영. App #117 Runtime Gate와 #112 Validation은 별도 Open 유지)
- 문서·Delivery 후속 현행화: **2026-09-29 KST** (App PR #128, GitOps PR #145/#146 merge 확인. PR #146의 GitOps Desired State 변경은 운영 Argo CD Sync·Pod Ready 재검증과 구분)
- UX Fix 후속 현행화: **2026-09-30 KST** (App #129 완료, PR #130·GitOps PR #149 병합 및 지정 Browser Evidence 연결. #112/#117과 cp-03/API VIP HA의 미완료 경계 유지)
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

위 표의 SHA는 최초 스냅샷 작성 기준이다. 후속 App PR #113은 2026-09-26 `59c9fe6cf723aecac6622e2ef8ea47884870dd25`로 병합됐다. 해당 문서 전용 PR은 Jenkins 필수 상태 미보고·서버 접근 불가로 예외적 bypass merge했으며, 이 SHA의 Jenkins PASS를 주장하지 않는다. 이후 상태는 해당 Merge와 Issue 종료 기록을 함께 읽는다.

작성 기준 `seokpan-app/main`의 Commit은 README 정리 Commit을 포함한다. 최종 Application 제품 변경은 Room 공통 2열 Layout까지 반영된 Source Freeze 계열이며, 이후 Jenkins Image Pipeline과 GitOps Promotion으로 실제 Desired State가 갱신됐다.

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

2026-09-30 후속 UX Fix [App #129](https://github.com/seokpan/seokpan-app/issues/129)는 닫힌 Room Socket에서 동작하지 않는 `상태 다시 확인` 버튼을 제거한 [PR #130](https://github.com/seokpan/seokpan-app/pull/130) 병합 후 완료됐다. 병합 Source `1128ebcc21bc1523aea68f46659ce6beeed7b00d`의 Frontend는 Jenkins Image Pipeline main Build #31 및 [GitOps Promotion PR #149](https://github.com/seokpan/seokpan-gitops/pull/149) 병합으로 Desired State에 반영됐고 Backend는 `NO_COMPONENT_CHANGE`였다.

App #129/#112에 기록된 `cp-01/root` 직접 기능 Gate `DIRECT_GATE_EXIT_CODE=0`과 Windows Host 실제 Gateway Browser의 무효 CTA 부재·수동 재연결·같은 Room Binding·교차 Chat·시험 Room/Session 정리 `FINAL_EXIT_CODE=0`, `BROWSER_EXIT_CODE=0`을 해당 Fix의 완료 근거로 연결한다. 이는 기존 실행 기록을 반영한 것으로 이번 문서 작업에서 Runtime을 재측정한 결과가 아니다. #112 V-04 전체 및 cp-03/API VIP HA 장애는 별도 미완료 범위이며 이 기능 Gate를 Cluster HA 전체 PASS로 확대하지 않는다.

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

App #123 / PR #124에서 Frontend Runtime Image의 `libexpat` 수정 가능한 HIGH를 패치했다. Jenkins Image Pipeline main Build #28은 Frontend `CRITICAL=0`, 수정 가능한 `HIGH=0`, Candidate Health·Promote·Final Digest를 확인했다. GitOps PR #141 병합 Revision `595288b25eff3b6a314eff3ce4b1be7444a09fdf`에서는 VM Argo `apps-frontend` Synced/Healthy, Frontend 2/2 Ready·Updated·Available와 Worker 분산, Gateway `/login` HTTP 200을 확인했다([App #123 완료 근거](https://github.com/seokpan/seokpan-app/issues/123)). 이 결과는 해당 Revision의 검증이며 App #112/#117의 미완료 범위를 해소하지 않는다.

후속 App PR #128은 Reference 문서와 Dockerfile 주석만 정리했다. 그 병합 Commit `0005d0d8c834405ad31a4eb4d0d166eb04768744`를 Jenkins Image Pipeline Build #29가 처리했고, Dockerfile 경로 변경으로 Backend·Frontend 두 Component의 새 Digest와 Source SHA를 기록한 GitOps Promotion PR #146이 승인 후 병합됐다. 이는 **GitOps Desired State 반영** 근거이며, 해당 Revision의 운영 Argo CD Sync·Pod Ready·Gateway Smoke를 이번 문서 현행화에서 재조회한 결과는 아니다. 단순 주석 변경도 현재 Component 영향 경로에는 포함될 수 있으므로 비영향 변경의 rollout 방지는 경로 판정 범위에서만 주장한다.

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

2026-09-23 Closeout에서 `seokpan-app#76/#86/#88/#85`의 구현 상태와 강화 Validation을 분리했고, 2026-09-26 PR #113 병합 후 #88 → #76 → #3을 종료했다.

현재 판정:

- `#76` — F01~F16을 Complete / Source Complete / NOT REQUIRED / Deferred / Validation Pending으로 재분류 완료
- `#86` — Game lifecycle Source 구현 완료, Issue completed
- `#88` — Realtime Source/정책 구현과 App 내부 정책 문서 정합화 PR #113 병합 완료, 구현 Issue 종료
- `#85` — Local Operation / Realtime Presentation 구현 완료, Issue completed
- 실제 Provider·2-Pod·Failure Boundary·Measurement는 `seokpan-app#112`가 Canonical Validation Backlog

특히 Game lifecycle은 다음 경계를 유지한다.

```text
Captured lifecycle Source = IMPLEMENTED / main 반영
Declared lifecycle mode = legacy (GitOps 설정·Application 기본값)
Captured production activation = NOT PERFORMED / DEFERRED
```

현재 `seokpan-gitops/apps/backend/configmap.yaml`에는 `SEOKPAN_GAME_LIFECYCLE_MODE`가 없고 Application 기본값은 `legacy`다. 따라서 captured lifecycle이 Production에서 활성화됐다고 주장하지 않는다. 이 문서는 2026-09-27 운영 Pod의 실효 환경변수를 재조회한 증거가 아니다.

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
- App PR #119에서 Realtime 10초 재접속 계약·Provider 문서를 추가 현행화해 main에 반영했다. PR #119의 Jenkins 실패는 기존 `undici 8.10.0` audit 취약점 때문이었고 Backend/Frontend 검사·빌드는 통과했다. 후속 PR #120에서 `undici 8.10.2`와 Background Runner transient Redis 복구 Source를 main에 반영했으며 Jenkins `pr-head`는 success였다. 다만 App #117의 실제 Runtime 복구 Gate와 #112 강화 Validation은 별도 Open 범위다.

### 7.3 MariaDB Backup 공유 Lock — 현행 구현 계약

Infra PR #168에서 공유 NFS lock은 초기 `flock -w 60` 대기 방식으로 검증됐으나 최종 리뷰에서 `flock -n -x 200`으로 바뀌었다. 현재 `backup_chain.sh.j2`는 잠금 경합 시 즉시 정상 스킵하고 다음 정기 cron까지 재시도하지 않는다. 60초 대기 및 NFS 폴링 관찰은 [DB-010](troubleshooting/database-storage-recovery/DB-010_MariaDB_백업_체인_상태_NFS_이전_공유_lock.md)의 당시 이력으로만 유지한다. 공유 상태의 동시 writer 방지는 PR #168의 코드·lock 실측 근거이며, 전체 백업 스크립트 동시 실행 실측 완료로 확대하지 않는다. 스킵된 주기의 Backup 미생성·RPO 영향을 운영 확인 대상으로 둔다. 실제 서버 배포 상태는 이번 문서 현행화에서 재조회하지 않았다.

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

2026-09-23 스냅샷에서는 아래 세 영역의 역할별 09/11/12를 확인했다.

- Kubernetes & Application Integration — 정태훈
- Database / Storage / Recovery — 김상희
- Delivery / Observability — 최유준

후속으로 Network / Ansible / External Infra — 이유빈 문서가 PR #178/#180/#182를 통해 추가됐다. 현재 역할별 09·11·12는 각각 네 영역이며, Network 문서는 다음 실제 파일을 사용한다.

| 문서 | 현재 파일 | 추가 근거 |
| --- | --- | --- |
| 09 실행·통합 실시설계 | [Network / Ansible / External Infra 09](09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Network_Ansible_External_Infra_이유빈.md) | [PR #178](https://github.com/seokpan/seokpan-docs/pull/178) |
| 11 구축·자동화 Runbook | [Network / Ansible / External Infra 11](11_MVP_구축·자동화_Runbook/11_MVP_구축·자동화_Runbook_Network_Ansible_External_Infra_이유빈.md) | [PR #180](https://github.com/seokpan/seokpan-docs/pull/180) |
| 12 검증·측정 계획 | [Network / Ansible / External Infra 12](12_MVP_검증·측정_계획/12_MVP_검증·측정_계획_Network_Ansible_External_Infra_이유빈.md) | [PR #182](https://github.com/seokpan/seokpan-docs/pull/182) |

PR #182의 병합 시각은 2026-09-28 15:19:45 UTC, 한국시간으로 **2026-09-29 00:19:45 KST**다. 이 문서 추가를 9월 23일 종료 시점에 이미 존재했던 것으로 소급하지 않는다.

Network 12의 `Partial / Blocked / Not Tested / Planned / Deferred`는 해당 문서와 실행 근거에서 그대로 추적한다. 파일 추가는 Network 전체 Acceptance 완료를 뜻하지 않는다. [Docs #89](https://github.com/seokpan/seokpan-docs/issues/89)는 Network / Ansible / External Infra 09·11·12와 CURRENT_STATE를 최종 교차검증한 뒤 2026-09-29 **A안(기존 1차 Source와 Evidence로 충분, 별도 Transition/Handoff 문서 불필요)**으로 completed 종료했다. 이 판단은 2차 상세 아키텍처나 기술 선택을 확정하지 않는다.

App / Infra / GitOps README의 2026-09-23 요약형 개편 이후, App PR #116(애플리케이션 구조·배포 안내), GitOps PR #136(배포 흐름·NetworkPolicy 서술), Infra PR #218(실제 VM 구성도)이 2026-09-28 각 저장소 main에 병합됐다. 이후 2026-09-29 App PR #119, Infra PR #220, GitOps PR #138 및 Docs PR #184로 현재 계약·실행 안내·배포 주석·Network 인덱스를 추가 현행화했다. App PR #128과 GitOps PR #145의 문구 현행화도 병합됐으며, Image Promotion PR #146의 병합 범위는 4.3절에서 구분한다. 문서 변경은 운영 VM·Cluster 상태를 새로 측정한 결과와 구분한다.

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
- `seokpan-app#117` — Background Runner transient Redis 복구 Source 반영 후 Runtime Gate
- `seokpan-gitops#57` — CD / Self-Heal / Rollback
- `seokpan-gitops#109` — F04 Runtime / Loki Evidence

### Closeout / Phase 2

- `seokpan-docs#153` — Current State Closeout
- `seokpan-docs#89` — 1차 → 2차 Handoff 구조 판단 (completed, A안: 기존 Source로 충분)
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
