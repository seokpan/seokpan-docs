# MVP 검증·측정 계획
## Kubernetes & Application Integration

## 1. 목적과 문서 경계

이 문서는 「石나가는 판단」 1차 프로젝트의 Kubernetes & Application Integration 영역에서 **무엇을 시험하고, 무엇을 측정하며, 어떤 근거로 PASS/FAIL을 판정할지** 정의한다.

```text
09 = What / Why / Responsibility / Gate
11 = Pre-check / Apply / Verify / Re-run / Recovery / Rollback
12 = Test Case / Preconditions / Stimulus·Fault / Measurement / PASS·FAIL / Evidence
```

12는 11의 실행 명령을 복제하지 않는다. 실제 실행 절차는 11을 참조하고, 본 문서는 각 Test의 선행조건·실행 자극/장애·관찰 항목·PASS/FAIL 기준·측정 Evidence를 정의한다.

직접 기준:

- [`02_SeokPan_핵심_문제_및_검증_목표.md`](../01-08_기획·설계_Baseline/02_SeokPan_핵심_문제_및_검증_목표.md)
- [`07_SeokPan_확장_호환형_MVP_도출.md`](../01-08_기획·설계_Baseline/07_SeokPan_확장_호환형_MVP_도출.md)
- [`09_MVP_실행·통합_실시설계_Kubernetes_Application_Integration_정태훈.md`](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Kubernetes_Application_Integration_정태훈.md)
- [`10_GitHub_협업_및_Repository_운영.md`](../10_GitHub_협업_및_Repository_운영/10_GitHub_협업_및_Repository_운영.md)
- [`11_MVP_구축·자동화_Runbook_Kubernetes_Application_Integration_정태훈.md`](../11_MVP_구축·자동화_Runbook/11_MVP_구축·자동화_Runbook_Kubernetes_Application_Integration_정태훈.md)
- [`PROJECT_CHANGES.md`](../PROJECT_CHANGES.md)
- [`MVP_IMPLEMENTATION_BASELINE.md`](../MVP_IMPLEMENTATION_BASELINE.md)
- `seokpan-app`, `seokpan-gitops`, `seokpan-infra` 최신 `main`, Issue, PR, Runtime Evidence

실제 Test Run 직전에는 관련 Repository의 최신 Revision과 열린 Issue/PR을 다시 확인한다.

---

## 2. 상위 검증축

02의 검증축을 대체하지 않는다.

| ID | 검증축 | 핵심 판정 원칙 | 본 역할 |
| --- | --- | --- | --- |
| M-01 | 동시성 정확성 | 정상 Turn Move 1개, Pass Turn Move 0개, 중복 Result/Rating 0건, stale 오염 0건 | 핵심 |
| M-02 | Vote 처리 성능 | 단계 부하로 처리량·p95/p99·오류율과 병목 Baseline 측정 | 핵심 |
| M-03 | Backend 장애 복구 | 상태 복원, 잘못된 사용자 패배 판정 0건, 복구시간 측정 | 핵심 |
| M-04 | DR | Backup/Restore·무결성·RTO·RPO | Cross-role |
| M-05 | Ansible 개선 | 수동 대비 구축·재구축 시간·개입·실패 Task | Cross-role |

추가 통합 Evidence:

```text
Application Artifact Traceability
Runtime Registry Pull / imagePullSecret Consumer
GitOps Sync / Self-Heal / Rollback
Frontend Runtime
HTTPRoute / HTTPS / WSS
Browser M5 First Success
Application Metrics / Logs
Config / Secret / CA Consumer 반영
```

---

## 3. 상태와 측정 원칙

### 3.1 상태

```text
Planned
Defined
Implemented
Merged
Running
Validated
Partial
Blocked
Deferred
Not Tested
Failed
```

다음을 같은 의미로 취급하지 않는다.

```text
Manifest 존재 ≠ Runtime Running
Runtime Running ≠ Integration Validated
Image Pushed ≠ GitOps Updated ≠ Argo Synced ≠ Pod Ready
Registry Pull Capability ≠ Actual Workload Pull
Platform Running ≠ Application Consumer Validated
```

### 3.2 임의 목표값 금지

다음은 실험 근거 없이 확정하지 않는다.

- 동시 사용자 수
- Vote events/s 목표
- p95/p99 목표 지연
- 허용 Error rate
- HPA CPU/Memory Target
- min/max Replicas
- RTO/RPO 목표값
- 개선률

실제 물리환경에서 부하를 단계적으로 올리고 급격한 지연·오류 증가 지점과 병목을 Baseline으로 정의한다.

### 3.3 현재 MVP 범위

현재 1차 MVP에는 별도 ANALYSIS Runtime을 배포하지 않는다. 현재 M-02 PASS/FAIL은 Vote/Game 핵심 경로를 기준으로 한다.

향후 ANALYSIS가 실제 Runtime에 포함되면 동일 부하 조건의 Before→After를 별도 Run으로 추가한다.

---

## 4. Evidence 규격

각 Run 최소 Metadata:

```text
Test Case ID
Run ID
Artifact Target — backend / frontend / common
Start / End Timestamp
Operator
App Commit SHA
GitOps Commit SHA
Infra Commit SHA
Backend / Frontend Digest
Kubernetes Context / Namespace
Related Issue / PR
Precondition State
Result State
Failure / Block Reason
Manual Steps
Follow-up
```

가능하면 함께 저장:

```text
11 Runbook Section / 실행 명령
명령 출력
Kubernetes Snapshot
Argo Sync / Health
Metric Export
Log Query
Load Generator Raw Result
Screenshot
Export Path
Checksum
DB Revision
Replication
Redis State
HTTP / WSS Result
Browser E2E
Recovery Timeline
RTO / RPO — 해당 Test에서 측정 시
```

금지:

```text
Password
전체 DB URL
Token
Private Key
kubeconfig Credential
Kubernetes Secret 실제 값
```

### 4.1 Run ID

```text
<TEST-CASE-ID>-YYYYMMDD-HHMM-<short-sha>
```

예:

```text
KAI-RUN-01-20260914-2030-a1b2c3d
```

### 4.2 실패 Run 보존

```text
실패 Run 보존
→ 원인·조치 기록
→ 새 Run ID
→ 재실행
```

FAIL을 수정해서 PASS로 덮어쓰지 않는다.

---

## 5. 정량 측정 Catalog

| 항목 | 단위 | 측정 방법/출처 | 용도 |
| --- | --- | --- | --- |
| Request latency | ms | Load generator 또는 Application request timing | p50/p95/p99 |
| Vote throughput | events/s | 성공 처리 Vote 수 / 측정 구간 초 | M-02 |
| Error rate | % | 실패 요청 / 전체 요청 × 100 | M-02 |
| Timeout count | count | Load result / App log | 병목 진단 |
| Recovery time | s | 장애/변경 시각 → 정상화 시각 | M-03/CD |
| Reconnect success | % | 성공 재연결 / 재연결 시도 | M-03 |
| Wrong loss decision | count | 장애 원인 오패배 판정 | M-03, 목표 0 |
| Move count | count/turn | DB/Event 상태 | M-01 |
| Duplicate Result/Rating | count | DB 비교 | M-01, 목표 0 |
| stale state mutation | count | request/state_version 로그·상태 비교 | M-01, 목표 0 |
| Replica count | count | Deployment/HPA status | Scale/HPA |
| CPU | cores/% | Prometheus / Metrics API | 진단 |
| Memory | MiB/GiB | Prometheus / Metrics API | 진단 |
| Scale time | s | HPA desired change → target Ready | HPA |
| Build/Push/Sync/Ready | s | Jenkins/PR/Argo/Pod Timestamp 차이 | Delivery |
| Prometheus target | UP/DOWN | Prometheus Target 상태 | Metrics |
| Log ingestion delay | s | App log event timestamp → Loki 조회 가능 시각 | Logs |
| RTO/RPO | s | Data/Storage Recovery 담당 Evidence | M-04 Cross-role |

측정 출처가 여러 개인 경우 Run Metadata에 실제 Source를 명시한다.

---

## 6. Dependency Evidence 재사용

Revision이 바뀌지 않은 검증 완료 Capability는 재사용할 수 있다.

예:

- Kubernetes / Calico / CoreDNS
- Namespace / RBAC
- Argo Root/Child Application 구조
- Redis StatefulSet/PVC Persistence
- DB Runtime/Migration Secret 공급 자동화
- MaxScale TLS Endpoint와 Root CA/서버 인증서 검증 경로
- One-shot Migration 자산 정적/API 검증
- Gateway Platform HTTPS/TLS
- Runtime pull-only Robot/Secret의 임시 Pod Digest Pull Evidence

재검증 조건:

```text
Manifest / Playbook / Image 변경
Version 변경
Secret / CA / Registry Credential 변경
Cluster 재구축
장애·복구 후
Consumer 최초 연결
Evidence Revision 불일치
```

특히 임시 Pod Pull PASS를 실제 Backend/Frontend Deployment Pull PASS로 대신하지 않는다.

---

## 7. 현재 검증 기준 상태

| 영역 | 상태 | 12 판정 |
| --- | --- | --- |
| Kubernetes / Calico / CoreDNS | Validated | 재사용 가능 |
| Namespace / RBAC / Argo CD | Validated | 재사용 가능 |
| Redis Runtime / Persistence | Validated | 재사용 가능 |
| Backend → Redis | 실제 Backend 연결 및 2 Replica Session 공유 확인 | Validated for A-10 |
| Backend/Frontend Child Application | Root 편입 / `Synced` / `Healthy` | Validated |
| Backend Runtime | 2 Replica, verified Digest, worker-01 / worker-02 분산 | Validated for A-10 |
| Frontend Runtime | 2 Replica, verified Digest | Validated for A-10 |
| A-10 Provider Integration | Completed | PASS |
| Backend Origin JSON | JSON 배열 형식 Runtime 반영 | PASS |
| Runtime pull-only Robot / Secret | Actual Workload에서 사용 | PASS |
| Deployment `imagePullSecrets` | Backend/Frontend Runtime 소비 | PASS |
| Actual Backend/Frontend Workload Pull | Verified Digest Pull / Ready | PASS |
| Migration Gate | 기존 Runtime DB + 신규 빈 DB 재현 | PASS |
| MaxScale TLS / CA Consumer | 실제 Backend TLS 1.3 Session 및 CA Fingerprint 확인 | PASS |
| Backend 2 Replica Shared Runtime | Scale-out/Session 공유 기본 경로 확인, Realtime Edge Case는 P4 진행 중 | Partial |
| HPA | Not Implemented | Not Tested |
| Application HTTPRoute | Running | PASS |
| HTTPS / API / WebSocket | A-10 Runtime Gate 통과 | PASS |
| ServiceMonitor | Runtime 활성화 | PASS |
| Application Metrics | Prometheus Target 2개 `UP`, 실제 Metric Query 확인 | PASS |
| Application Logs | Platform 수집 경로 존재, 본 역할의 최종 App Log Evidence 미확정 | Partial / Not Final |
| Argo CD Self-Heal | 실제 Live State Drift 복구 확인 | PASS |
| Git Revert Rollback | 실제 Runtime Rollback Run 미수행 | Not Tested |
| P4 Stabilization / Acceptance | `seokpan-app#76`에서 진행 중 | In Progress |

A-10 완료와 P4 Acceptance 완료를 같은 의미로 사용하지 않는다.

## 8. Test Traceability Matrix

| Test Case | 목적 | Gate | 11 | 현재 상태 |
| --- | --- | --- | --- | --- |
| KAI-PRE-01 | Dependency Snapshot | A~C | 4~6 | Validated |
| KAI-DEL-01 | Artifact / Registry / GitOps / Argo | B~D/E | 5/8/11/13 | PASS |
| KAI-MIG-01 | Migration Gate | C→D | 7 | PASS |
| KAI-RUN-01 | Backend 1 Replica Provider | D | 8 | PASS |
| KAI-RUN-02 | Backend 2 Replica Shared Runtime | E | 9 | Partial — A-10 Scale-out PASS, P4 Realtime 세부 검증 진행 중 |
| KAI-HPA-01 | HPA / Workload Distribution | E | 10 | Not Implemented / Not Tested |
| KAI-FE-01 | Frontend Runtime | E | 11 | PASS |
| KAI-RT-01 | HTTPRoute / HTTPS / WSS | F | 12 | PASS |
| KAI-OBS-01 | Application Metrics | G | 14 | PASS |
| KAI-OBS-02 | Application Logs | G | 14 | Partial / 최종 Evidence 미확정 |
| KAI-E2E-01 | Browser M5 First Success | F~G | 11~14 | In Progress — 외부 접속 Runtime Gate PASS, P4 전체 사용자 흐름 미완료 |
| KAI-CON-01 | M-01 / P4 | G | 9 | In Progress / Not Final |
| KAI-PERF-01 | M-02 / P4 | G | 9~10 | Not Tested |
| KAI-REC-01 | M-03 / P4 | G | 9/15/17 | In Progress / Not Final |
| KAI-CD-01 | Self-Heal | G | 13 | PASS |
| KAI-CD-02 | Git Revert Rollback | G | 13/17 | Not Tested |
| KAI-CFG-01 | Config / Secret / CA Consumer | D~G | 16~17 | PASS for DB Secret / Root CA consumer path |
| KAI-DR-01 | Restore 후 Application 정상화 | G | 17 | Not Tested / Cross-role |

`PASS`는 해당 Test Case의 현재 정의 범위에 실제 Runtime Evidence가 존재할 때만 사용한다.

`KAI-RUN-02`는 Backend 2 Replica 배포 자체가 성공했다는 이유만으로 전체 PASS 처리하지 않는다. Cross-Pod WebSocket, Chat 권한, Runner/Realtime lifecycle과 reconnect 수렴은 P4에서 계속 검증한다.

## 9. Test Contract Matrix

| Test | Preconditions | Stimulus / Fault | Observation | PASS 핵심 | Evidence |
| --- | --- | --- | --- | --- | --- |
| KAI-PRE-01 | 대상 Revision 식별 | 없음, Snapshot | Node/Argo/Secret/CA/Image | Dependency와 Revision 일치 | Snapshot/Commit/Digest |
| KAI-DEL-01 | 승인 Artifact, Runtime Pull Capability | GitOps Digest+imagePullSecrets 반영 | Pull/Pod ImageID/Ready | Commit→Digest→Actual Pod 일치 | Jenkins/Harbor/Infra/GitOps/Pod |
| KAI-MIG-01 | DB/Secret/CA/Backup/Replication | current 또는 승인 Mutation | Revision/Job/Replication/Data | 기대 Revision·데이터·복제 정상 | Approval/Job/BeforeAfter |
| KAI-RUN-01 | A-10, Delivery, Migration PASS | replicas 1 활성화 | Probe/DB/Redis/Readiness | 실제 Provider로 1 Pod Ready | Pod/Log/Connection |
| KAI-RUN-02 | RUN-01 PASS | replicas 2 | Shared state/PubSub/Runner/Reconnect | Replica 간 상태 수렴, 중복 Runner 없음 | Pod/Redis/Timeline |
| KAI-HPA-01 | 2 Replica, Metrics, `replicas` 관리 주체 확정 | 단계 부하 | HPA/Replica/latency/error | Scale 정상, Argo CD와 `replicas` 변경 충돌 없음 | HPA/Metric/Argo |
| KAI-FE-01 | Frontend Artifact | Frontend 활성화 | Pod/Service/SPA | 승인 Digest Runtime 정상 | Pod/HTTP |
| KAI-RT-01 | Backend+Frontend+Gateway | HTTPRoute 적용 | Conditions/HTTPS/WSS/CORS | Route·HTTPS·WSS 정상 | Route/HTTP/WSS |
| KAI-OBS-01 | `/metrics` 제공 및 Service label이 ServiceMonitor selector와 일치 | ServiceMonitor 활성화 | Target/Scrape/Metric | Target UP, 실제 Metric 조회 | Prometheus |
| KAI-OBS-02 | App Pod Running | 기준 Log Event 발생 | Alloy/Loki/labels/delay | 실제 App Log 조회, label 일치 | Log Query |
| KAI-E2E-01 | M3 Runtime + M4 Delivery/Observability 선행조건 충족 | 팀원 PC 실제 사용자 흐름 | Browser/Network/App state | 실제 URL/Provider/데이터 UX 정상 | Screenshot/Trace/Log |
| KAI-CON-01 | Shared Runtime | 동시 Vote/stale/retry | Move/Result/Rating/state | M-01 0건 조건 | DB/Redis/Log |
| KAI-PERF-01 | MVP Runtime | 단계 부하 | events/s, latency, error, resource | 측정 완결, M-01 유지, 병목 식별 | Raw load/Metric |
| KAI-REC-01 | 2 Replica/Reconnect | 단일 Backend 장애 | reconnect/state/recovery/wrong-loss | 상태 복원, 오패배 0 | Timeline/State |
| KAI-CD-01 | 검증 완료된 Git Desired State | 안전한 Live Drift(실행 중 리소스를 일시적으로 Git과 다르게 변경) | OutOfSync/SelfHeal | Git 상태로 복구 | Argo/BeforeAfter |
| KAI-CD-02 | 정상 동작이 검증된 Git Revision | Git Revert | Sync/Ready/Recovery | 검증된 Revision으로 복구 | PR/Argo/Timeline |
| KAI-CFG-01 | 승인 Config 변경 | Config/Secret/CA 변경 | Pod UID/new config/readiness | 새 Pod가 변경된 Config/Secret/CA 값을 실제로 사용 | Revision/Pod/Fingerprint |
| KAI-DR-01 | Restore 수행 | 복원 후 Consumer 연결 | Data sample/readiness/browser | 무결성+App 정상화 | DR+App Evidence |

---

## 10. Test 상세 판정

### KAI-DEL-01 — Artifact / Registry Pull / GitOps / Argo

Traceability:

```text
App Commit
→ Jenkins Run
→ Harbor Digest
→ Runtime pull-only Robot
→ application/harbor-pull-secret
→ Deployment imagePullSecrets + Digest
→ GitOps PR/Merge
→ Argo Sync
→ Actual Pod Pull
→ Pod ImageID / Ready
```

Capability와 Consumer를 분리한다.

A-10에서는 임시 Pull Capability뿐 아니라 실제 Backend/Frontend Deployment가 Runtime 전용 `imagePullSecret`과 검증 Digest를 소비해 Pod `Ready`까지 도달했다.

전체 PASS:

- Runtime Robot이 pull-only.
- application Namespace 전용 Secret 사용.
- Deployment가 Secret을 명시적으로 소비.
- 검증 Digest 고정.
- Actual Pod ImageID가 Digest와 일치.
- ImagePull 성공 및 Workload Ready.
- `latest`/`git-pending` 실제 Runtime 미사용.

현재 결과:

```text
Registry Pull Capability: PASS
Actual Backend/Frontend Workload Pull: PASS
Verified Digest / Pod ImageID / Ready: PASS
KAI-DEL-01: PASS
```

Evidence:

- `seokpan-gitops#57` — GitOps Digest Pinning, 실제 Workload Pull, Argo CD Runtime 배포 경로 검증
- `seokpan-app#3` — A-10 Runtime이 실제 소비한 Backend/Frontend Digest와 Deployment 상태 연결

### KAI-MIG-01 — Migration Gate

공통 Preconditions:

- 승인 Backend Digest
- Migration Secret / CA
- Replication 정상
- Backup/Restore 준비
- Active Mutation 없음

Read-only `current`는 Mutation 승인 Reference를 요구하지 않는다.

`stamp-baseline`/`upgrade-head`는 승인 Reference가 필요하다.

PASS:

- Audit와 Action 일치.
- 기대 Revision.
- Replication 정상.
- 표본 데이터 보존.
- Runtime/Migration Credential 경계 유지.
- Mutation 시 승인된 Job만 1회 성공.

현재 결과: `PASS`.

Evidence:

- `seokpan-app#22` — 기존 Runtime DB Audit/Stamp/후속 Revision, 기존 데이터 보존·Replication, 신규 빈 DB `alembic upgrade head` 재현

### KAI-RUN-01 — Backend 1 Replica

Preconditions:

- A-10 완료
- Backend KAI-DEL-01 실제 Workload Pull PASS
- KAI-MIG-01 PASS
- GitOps #90 완료
- Runtime Secret/CA/Redis

PASS:

- 승인 Digest의 Backend Pod 1 Ready.
- Startup/Live/Ready 정상.
- DB TLS/Identity/Game 연결 성공.
- Redis Consumer 성공.
- Migration Credential 미소비.
- Memory Provider fallback 없음.

현재 결과: `PASS`.

Evidence:

- GitOps PR #93 — Backend 1 Replica 실제 Runtime 활성화
- `seokpan-app#50` — MaxScale TLS를 통한 Identity/Game DB 실제 Session
- `seokpan-gitops#7` — Redis Service DNS를 통한 실제 Backend Consumer 연결

### KAI-RUN-02 — Backend 2 Replica

현재 A-10 Scale-out 결과:

- Backend Pod 2개 `Ready`.
- worker-01 / worker-02 분산.
- 두 Pod가 동일한 검증 Backend Digest 사용.
- Redis 기반 Session 공유 기본 경로 확인.

현재 판정: `Partial`.

아직 P4에서 검증 중인 항목:

- Cross-Pod WebSocket replacement.
- Room WebSocket과 Chat이 서로 다른 Pod를 타는 경우의 권한 확인.
- Runner 오류가 Realtime 전체 가용성에 미치는 영향.
- reconnect/disconnect lifecycle 수렴.
- Pub/Sub 누락 이후 Snapshot 복구의 세부 Edge Case.

전체 PASS 기준:

- Pod 2개 `Ready`.
- 공유 Runtime State가 Replica 간 일관되게 수렴.
- 특정 Backend Pod의 Process Memory가 Replica 간 공유해야 하는 상태의 Source of Truth(현재 상태를 판단하는 기준 저장소)가 되지 않음.
- Pub/Sub 누락/재접속 시 Snapshot으로 수렴.
- Runner 중복 마감 없음.

### KAI-HPA-01

현재 Not Implemented / Not Tested.

PASS:

- Metrics 정상.
- 설정 범위 내 Replica 변화.
- Scale 중 가용성 유지.
- Argo/HPA replicas 경쟁 없음.
- M-01 오류 없음.

임계값 자체는 실제 단계 부하 결과로 판단한다.

### KAI-FE-01

현재 결과: `PASS`.

Evidence:

- `seokpan-app#3` — Frontend 2 Replica Runtime 완료 상태
- GitOps PR #96 — Frontend 2 Replica와 Same-Origin Gateway 경로 반영

Preconditions:

- Backend Scale/HPA 단계 판정
- Frontend KAI-DEL-01 Actual Pull PASS
- Container Smoke

HPA 구현을 이번 MVP 범위에서 보류(Deferred)하면 PASS로 처리하지 않고, 구현하지 않기로 한 이유와 Go/No-Go 판단 근거를 남긴다.

PASS:

- 승인 Digest 실행.
- Pod/Service 정상.
- SPA 기본 경로 정상.

### KAI-RT-01

현재 결과: `PASS for A-10 external Runtime Gate`.

Evidence:

- `seokpan-infra#188` — Let's Encrypt Production TLS, Windows Host/Linux VM HTTPS 접속, Guest Session/Lobby WebSocket 확인
- `seokpan-app#3` — Gateway HTTPS/API/WebSocket Runtime Gate 완료 상태 연결

경로:

```text
/       → frontend:8080
/api/v1 → backend:8000
/ws/v1  → backend:8000
```

PASS:

- Accepted / ResolvedRefs 정상.
- HTTPS Frontend 성공.
- API Routing 성공.
- WSS handshake/message 성공.
- Cookie/CORS/Forwarded Header 정상.
- 내부 Health/Metric 의도치 않은 외부 공개 없음.

### KAI-OBS-01

현재 결과: `PASS`.

Evidence:

- `seokpan-app#78` — Backend `/metrics` 구현과 검증 Image 생성
- `seokpan-gitops#91` — ServiceMonitor Runtime 활성화, Backend 2 Replica Prometheus Target `UP`, 실제 Application Metric Query 확인

PASS:

- Backend `/metrics` 제공.
- Service metadata.labels와 ServiceMonitor selector 정합.
- Prometheus Target UP.
- 실제 App Metric 조회.

### KAI-OBS-02

PASS:

- Backend/Frontend stdout/stderr가 Loki에서 조회.
- namespace/pod/container/node_name 일치.
- 민감정보 Raw Log 노출 없음.
- Log ingestion delay 측정 가능.

Alert firing/resolved/E-mail은 Delivery/Observability 담당 주 Owner이며 최종 Acceptance에 연결한다.

### KAI-E2E-01 — Browser M5 First Success

Preconditions:

- Backend/Frontend Actual Workload Pull PASS
- Backend Runtime PASS
- Frontend Runtime PASS
- HTTPRoute/HTTPS/WSS PASS
- 실제 DB/Redis Provider
- Artifact Traceability
- 현재 M4 Delivery/Observability 선행 Evidence

Stimulus:

```text
팀원 PC
→ 실제 공개 URL 접속
→ 최신 MVP_IMPLEMENTATION_BASELINE의 구현 완료 핵심 사용자 흐름 수행
```

PASS:

- HTTPS/WSS.
- Same-Origin.
- 실제 Provider.
- 실제 데이터 UX/UI 정상.
- Fake/Memory E2E와 구분.

### KAI-CON-01 — M-01 / P4

PASS:

```text
정상 Turn Move = 1
Pass Turn Move = 0
중복 GameResult = 0
중복 Rating = 0
stale 잘못된 상태 변경 = 0
동일 idempotency key에 대한 최종 상태 반영 횟수 = 1
```

Backend-only 사전시험은 가능하지만 최종 P4는 M5 이후 동일 MVP Runtime 계열에서 재실행한다.

### KAI-PERF-01 — M-02 / P4

Stimulus:

```text
낮은 부하
→ 단계 증가
→ 각 단계 안정 측정
→ 급격한 악화 지점 확인
```

PASS 의미:

- 단계별 throughput/latency/error 측정 완료.
- M-01 정확성 유지.
- 병목 단계/원인 진단 가능.
- 근거 없는 목표/개선률 미생성.

최종 P4 Baseline은 M5 이후 실제 MVP Runtime에서 측정한다.

### KAI-REC-01 — M-03 / P4

Fault:

```text
검증된 2 Replica 중 단일 Backend 장애 주입
```

PASS:

- 사용자 몰수패/공동패배 오판 0.
- 재연결 성공.
- Room/Game/Turn/Board의 기준 상태가 장애 전 정상 상태로 복원.
- Recovery time 측정 가능.

### KAI-CD-01

Self-Heal 검증에서는 영구 데이터에 영향을 주지 않는 Live Drift(실행 중 Kubernetes 리소스를 Git Desired State와 일시적으로 다르게 만드는 변경)만 사용한다.

현재 결과: `PASS`.

Evidence:

- `seokpan-gitops#57` — Kubernetes Live State Drift를 의도적으로 만든 뒤 Argo CD Self-Heal이 Git Desired State로 복구한 Runtime 검증

PASS:

- OutOfSync 감지.
- Self-Heal 후 Git Desired State 수렴.
- 불필요한 가용성 영향 없음.

### KAI-CD-02

DB Schema Migration은 대상이 아니다.

PASS:

- 정상 동작이 이미 검증된 Git Revision으로 복귀.
- Argo Sync.
- Runtime Ready.
- Recovery time/manual steps 측정.
- 장기 Cluster patch 없음.

### KAI-CFG-01

이 Test만을 위해 운영 Credential/CA를 임의 회전하지 않는다.

현재 결과: `PASS for DB Secret / Root CA consumer path`.

Evidence:

- `seokpan-app#50` — Runtime DB Secret과 공개 Root CA를 소비한 실제 MaxScale TLS 연결
- `seokpan-gitops#39` — Root CA 갱신 후 새 Backend Pod의 Mount Fingerprint와 실제 DB TLS 연결 확인

PASS:

- 필요한 경우 새 Pod 생성.
- 새 Pod가 최신 Config/Secret/CA 값을 실제로 사용.
- CA 변경 시 기대 Fingerprint.
- Git 기록 없는 restart만으로 종료하지 않음.

### KAI-DR-01

M-04 Backup/Restore·RTO/RPO는 Data/Storage Recovery 담당이 주 Owner다.

PASS:

- Restore Evidence와 App Consumer Evidence가 동일 Run/Revision으로 연결.
- 표본 데이터 무결성 PASS.
- Application 정상화.

---

## 11. 검증 Phase

### Phase 0 — Registry / Artifact

현재 결과: `Completed`.

```text
Runtime Pull Capability
→ Backend/Frontend imagePullSecrets
→ Verified Digest
→ Argo Sync
→ Actual Workload Pull
→ Pod Ready
```

### Phase 1 — Backend First Runtime

현재 결과: `Completed`.

```text
A-10 Production Provider
→ KAI-MIG-01 PASS
→ KAI-RUN-01 PASS
```

### Phase 2 — Scale / HPA

현재 상태:

```text
KAI-RUN-02
├─ Backend 2 Replica Scale-out: A-10 Runtime 범위 PASS
└─ Shared Runtime 세부 검증: P4 진행 중

KAI-HPA-01
└─ Not Implemented / Not Tested
```

HPA는 Backend 2 Replica 이후 수행할 별도 Validation Track이다.

HPA 미구현은 Frontend / External Runtime 진행을 차단하지 않으며, HPA를 구현·측정하지 않은 상태에서 HPA PASS를 기록하지 않는다.

### Phase 3 — Frontend / External / Observability

현재 결과:

```text
KAI-FE-01: PASS
KAI-RT-01: PASS
Public TLS / External Access: PASS
KAI-OBS-01 Metrics: PASS
KAI-OBS-02 Logs: Partial / Final Evidence pending
```

Gateway Platform PASS를 Application Route PASS로 대체하지 않는다.

### Phase 4 — M5 First Success

현재 상태:

```text
External Runtime Gate
- Windows Host / Linux VM HTTPS 접속: PASS
- API / WebSocket 경로: PASS

KAI-E2E-01
- 전체 Browser 사용자 흐름 Acceptance: In Progress
```

최종 M5 First Success는 팀원 PC의 실제 URL에서 핵심 사용자 흐름과 실제 Provider/Data까지 확인했을 때 PASS로 판정한다.

외부 접속 성공만으로 KAI-E2E-01 전체를 PASS 처리하지 않는다.

### Phase 5 — MVP P4

M5가 성립한 동일 MVP Runtime 계열에서:

```text
KAI-CON-01
KAI-PERF-01
KAI-REC-01
KAI-CD-01 / KAI-CD-02
KAI-CFG-01 — 실제 변경 시
Cross-role KAI-DR-01
```

사전 Backend-only 결과는 결함 탐지 Evidence이며 P4 최종 결과를 대신하지 않는다.

### Absolute Gates

이미 통과한 A-10 선행 Gate를 과거 미완료 상태로 되돌려 기록하지 않는다.

현재 P4에서 여전히 유효한 절대 조건:

```text
Backend 2 Replica 세부 Shared Runtime 검증 미완료
→ M-01/M-03 최종 판정 금지

HPA 미구현
→ HPA Scale 결과 PASS 금지

Git Revert Rollback 미실행
→ KAI-CD-02 PASS 금지

P4 핵심 사용자 흐름 안정화 미완료
→ Final MVP Acceptance PASS 금지
```

---

## 12. Before → Change/Fault → After

### Replica / HPA

```text
Static Replica
→ Scale-out/HPA
→ 동일 부하 재실행
```

비교: events/s, p95/p99, error, resource, replica, M-01 오류.

### Backend 장애

```text
정상 Snapshot
→ 단일 Backend Fault
→ Reconnect/Recovery Snapshot
```

비교: 상태 일관성, recovery time, 실패 요청, 오패배.

### GitOps Rollback

```text
정상 동작이 검증된 상태
→ 검증 가능한 Git Revision 또는 안전한 Live Drift
→ Revert + Argo Sync
```

비교: recovery time, manual steps, Sync/Health, Pod Ready.

### Artifact / Registry

```text
App Commit
→ Build/Push
→ Runtime Pull Credential
→ GitOps Digest/imagePullSecrets
→ Argo Sync
→ Actual Pod Pull/Ready
```

---

## 13. 중단 기준

즉시 중단:

- 데이터 손실 위험
- 예상 외 DB Revision
- Replication 오류
- Secret/Token/Password 노출
- Runtime에 CI Push Credential 사용
- 의도하지 않은 Image Digest
- 잘못된 GameResult/Rating
- M-01 정확성 오류
- Backend 장애가 사용자 패배로 잘못 확정
- HPA/Argo replicas 지속 경쟁

중단 후:

```text
Run = FAILED / BLOCKED
→ 원인 기록
→ 영향 범위 고정
→ 11 Rollback/Recovery
→ 새 Run ID 재실행
```

---

## 14. 결과 기록 Template

```text
Test Case:
Run ID:
Artifact Target:
Date/Time:
Operator:
State: PASS / FAIL / BLOCKED / NOT TESTED

App Commit:
GitOps Commit:
Infra Commit:
Backend Digest:
Frontend Digest:

Preconditions:
Stimulus/Fault:
Observation:
Measured:
PASS/FAIL Basis:

Evidence:
- Issue/PR:
- Runbook/Command:
- Log:
- Metric/Export:
- Screenshot:
- Snapshot/Checksum:

Failure/Exception:
Manual Steps:
Cleanup/Rollback:
Follow-up:
```

---

## 15. MVP Acceptance

Gate G는 본 역할만으로 완료되지 않는다.

```text
Kubernetes / Application Integration
+ Data / Storage Recovery
+ Delivery / Observability
+ Network / External Infra
+ Browser / E2E
```

### 15.1 현재 본 역할 검증 상태

| 항목 | 현재 상태 |
| --- | --- |
| Commit → Digest → Pull Secret → GitOps → Argo CD → Actual Pod Traceability | PASS |
| Migration Gate | PASS |
| Backend 1 Replica Provider Runtime | PASS |
| Backend 2 Replica Scale-out / Worker 분산 | PASS |
| Backend 2 Replica Shared Runtime 전체 | Partial — P4 세부 검증 진행 중 |
| HPA | Not Implemented / Not Tested |
| Frontend 2 Replica Runtime | PASS |
| HTTPRoute / HTTPS / API / WebSocket / Public TLS | PASS |
| Application Metrics | PASS |
| Application Logs 최종 Consumer Evidence | Partial |
| Browser External Runtime Gate | PASS |
| Browser 전체 M5 / 실제 데이터 UX Acceptance | In Progress |
| M-01 Concurrency | In Progress / Not Final |
| M-02 Performance Baseline | Not Tested |
| M-03 Recovery | In Progress / Not Final |
| Argo CD Self-Heal | PASS |
| Git Revert Rollback | Not Tested |
| Config / DB Secret / Root CA Consumer | PASS for verified path |
| DR 이후 Application 정상화 | Not Tested / Cross-role |

### 15.2 최종 MVP Acceptance에 필요한 남은 검증

최종 Gate G PASS를 위해 본 역할 관점에서 남은 핵심 항목은 다음과 같다.

- Backend 2 Replica Shared Runtime 세부 검증 완료
- Browser M5 First Success와 실제 Provider/Data 기반 사용자 흐름 PASS
- M-01 P4 최종 판정
- M-02 대표 부하 Performance Baseline
- M-03 장애·복구 최종 판정과 Recovery time 측정
- Application Log 최종 Consumer Evidence
- Git Revert 기반 실제 Rollback Run
- 필요한 경우 HPA의 구현 여부와 미구현/Deferred 근거 확정
- Cross-role DR 이후 Application 정상화 Evidence 연결

이미 PASS한 A-10 Runtime Integration 항목을 다시 미완료로 표현하지 않으며, 아직 실행하지 않은 P4/Rollback/DR/HPA 항목을 완료된 결과처럼 기록하지 않는다.

Alert Evidence와 DR Evidence는 Cross-role 결과로 동일 Acceptance Run에 연결한다.

## 16. 문서 완료 기준

- 02 M-01~M-05 Traceability.
- 07 Commit→Digest→Healthy 및 Metric/Log/Evidence 원칙과 정합.
- 09 Critical Path, M5, P4 순서 유지.
- 11 실행절차 반복 없이 Test Contract로 연결.
- Infra #180/PR #185와 GitOps #57 최신 Runtime Pull 경계 반영.
- Registry Capability와 Actual Workload Consumer 구분.
- 각 Test의 Preconditions / Stimulus·Fault / Observation / PASS·FAIL / Evidence 명확.
- 정량 항목의 단위와 측정 출처 명확.
- 미실행 항목을 결과처럼 작성하지 않음.
- 임의 성능/HPA/RTO/RPO 목표 금지.
- ANALYSIS Runtime을 현재 MVP PASS 필수조건으로 만들지 않음.
- Cross-role Owner 경계 명확.
- Secret/Token/Private Key Evidence 금지.
- 실패 Run 보존, 재실행은 새 Run ID.
- 실제 Repository/Runtime과 재대조 후 추가 필수 보완사항이 없을 때 종료.
