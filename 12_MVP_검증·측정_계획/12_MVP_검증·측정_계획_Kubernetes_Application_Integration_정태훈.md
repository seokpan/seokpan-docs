# MVP 검증·측정 계획
## Kubernetes & Application Integration

## 1. 목적과 문서 경계

이 문서는 「石나가는 판단」 1차 프로젝트의 Kubernetes & Application Integration 영역에서 **무엇을 시험하고, 무엇을 측정하며, 어떤 근거로 PASS/FAIL을 판정할지** 정의한다.

```text
09 = What / Why / Responsibility / Gate
11 = Pre-check / Apply / Verify / Re-run / Recovery / Rollback
12 = Test Case / Measurement / PASS·FAIL / Evidence
```

12는 11의 명령과 실행 절차를 반복하지 않는다. 실제 실행은 11을 참조하고 본 문서는 시험 목적·조건·측정값·판정 기준·Evidence를 소유한다.

상위 기준:

- [`02_SeokPan_핵심_문제_및_검증_목표.md`](../02_SeokPan_핵심_문제_및_검증_목표.md)
- [`07_SeokPan_확장_호환형_MVP_도출.md`](../07_SeokPan_확장_호환형_MVP_도출.md)
- [`09_MVP_실행·통합_실시설계_Kubernetes_Application_Integration_정태훈.md`](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Kubernetes_Application_Integration_정태훈.md)
- [`10_GitHub_협업_및_Repository_운영.md`](../10_GitHub_협업_및_Repository_운영.md)
- [`11_MVP_구축·자동화_Runbook_Kubernetes_Application_Integration_정태훈.md`](../11_MVP_구축·자동화_Runbook/11_MVP_구축·자동화_Runbook_Kubernetes_Application_Integration_정태훈.md)
- [`PROJECT_CHANGES.md`](../PROJECT_CHANGES.md)
- [`MVP_IMPLEMENTATION_BASELINE.md`](../MVP_IMPLEMENTATION_BASELINE.md)
- `seokpan-app`, `seokpan-gitops`, `seokpan-infra` 최신 `main`, Issue, PR, Runtime Evidence

실제 Test 실행 직전에는 관련 Repository의 최신 Revision과 열린 Issue/PR을 다시 확인한다.

---

## 2. 상위 검증축과 역할 연결

02의 검증축을 대체하지 않는다.

| 상위 ID | 검증축 | 핵심 판정 원칙 | 본 역할과의 관계 |
| --- | --- | --- | --- |
| M-01 | 동시성 정확성 | 정상 Turn Move 1개, Pass Turn Move 0개, 중복 Result/Rating 0건, stale 요청 오염 0건 | 핵심 책임 |
| M-02 | Vote 처리 성능 | 단계 부하로 처리량·p95/p99·오류율 변화와 병목 Baseline 확인 | 핵심 책임 |
| M-03 | Backend 장애 복구 | 상태 복원, 잘못된 사용자 패배 판정 0건, 복구시간 측정 | 핵심 책임 |
| M-04 | DR | Backup/Restore·데이터 무결성·RTO·RPO | Data/Storage Recovery와 Cross-role |
| M-05 | Ansible 개선 | 수동 대비 구축·재구축 시간·직접 개입·실패 Task 비교 | Network/Infra Automation과 Cross-role |

본 역할은 M-04/M-05의 주 측정 Owner가 아니다. 복구 후 Application Consumer 정상화 등 통합 Evidence를 제공한다.

### 2.1 추가 통합 검증축

M-01~M-05 외에도 07/09/11에서 요구하는 다음 통합 Evidence를 추적한다.

```text
Application Artifact Traceability
GitOps Sync / Self-Heal / Rollback
Frontend Runtime
HTTPRoute / HTTPS / WSS
Browser First Success
Application Metrics / Logs
Config / Secret / CA Consumer 반영
```

---

## 3. 측정 원칙

### 3.1 상태 구분

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
Platform Running ≠ Application Consumer Validated
```

### 3.2 근거 없는 목표값 금지

다음은 실험 근거 없이 임의 확정하지 않는다.

- 동시 사용자 수
- Vote events/s 목표
- CPU/Memory HPA Target
- `minReplicas` / `maxReplicas`
- p95/p99 목표 지연
- 허용 오류율
- RTO/RPO 목표값
- 개선률

성능은 부하를 단계적으로 증가시키고 지연·오류가 급격히 악화되는 지점과 자원 병목을 Baseline으로 정의한다.

### 3.3 핵심 KPI와 진단 지표

핵심 KPI:

```text
동시성 오류 건수
Vote 처리량
p95 / p99 지연
Error rate
Backend 장애 복구시간
잘못된 사용자 패배 판정 건수
RTO / RPO
```

보조 진단지표:

```text
CPU / Memory / Disk / Network
Pod / Replica 수
Backend별 요청 분포
WebSocket 연결 수
Active Room / Game
Redis / DB 연결 상태
HPA desired/current replicas
Argo Sync / Health
Prometheus Target
Loki Log ingestion
```

### 3.4 현재 MVP 범위 우선

현재 1차 MVP에서는 별도 ANALYSIS Runtime을 배포하지 않는다. 따라서 현재 M-02 PASS/FAIL은 Vote/Game 핵심 경로를 기준으로 한다.

향후 ANALYSIS가 실제 Runtime에 포함되면 동일 부하 조건에서 분석 비활성/활성 Before→After를 별도 Run으로 추가한다.

---

## 4. Evidence 규격

각 Run은 최소 다음을 기록한다.

```text
Test Case ID
Run ID
Artifact Target — 해당 시 backend / frontend
실행 시작/종료 시각
Operator
App Commit SHA
GitOps Commit SHA
Infra Commit SHA
Backend / Frontend Image Digest
Kubernetes Context / Namespace
관련 Issue / PR
선행조건 상태
State: PASS / FAIL / BLOCKED / NOT TESTED
실패 또는 중단 이유
수동 개입 단계
후속 조치
```

가능하면 다음을 함께 저장한다.

```text
실행 명령 또는 11 Runbook 참조
명령 출력
Kubernetes Object Snapshot
Argo CD Sync / Health
Metric Export
Log Query 결과
Load Generator 결과
Screenshot
Export 파일 경로
Checksum
DB Revision
Replication 상태
Redis 상태
HTTP / WSS 결과
Browser E2E 결과
Recovery Timeline
RTO / RPO — 해당 시나리오에서 측정하는 경우
```

### 4.1 Evidence 금지

다음은 저장하지 않는다.

- Password
- 전체 DB URL
- Token
- Private Key
- kubeconfig Credential
- Kubernetes Secret 실제 값

Secret은 Object 존재·Key 이름·Consumer 참조까지만 기록한다.

### 4.2 Run ID

Test Case ID 자체가 `KAI-` Prefix를 포함하므로 다음 형식을 사용한다.

```text
<TEST-CASE-ID>-YYYYMMDD-HHMM-<short-sha>
```

예:

```text
KAI-RUN-01-20260914-2030-a1b2c3d
```

동일 Test Case를 Backend/Frontend 등 서로 다른 Artifact에 적용하면 Metadata의 `Artifact Target`으로 구분한다.

### 4.3 실패 Run 보존

```text
실패 Run 보존
→ 원인/조치 기록
→ 새 Run ID 발급
→ 동일 Test Case 재실행
```

FAIL Run을 수정해 PASS로 덮어쓰지 않는다.

---

## 5. Dependency Evidence 재사용

관련 Revision이 바뀌지 않은 검증 완료 Platform Capability는 불필요하게 재구축하지 않는다.

재사용 가능 예:

- Kubernetes Cluster / Calico / CoreDNS
- Namespace / RBAC
- Argo CD Root/Child Application 구조
- Redis StatefulSet / PVC Persistence
- Backend DB Runtime/Migration Secret 공급 자동화
- MaxScale TLS Listener / CA 계약
- One-shot Migration 자산의 정적/API 검증
- Gateway Platform HTTPS/TLS

재검증 조건:

```text
관련 Manifest / Playbook / Image 변경
Version 변경
Secret / CA 변경
Cluster 재구축
장애/복구 후
실제 Consumer 연결 최초 수행
Evidence 대상 Revision과 현재 Revision 불일치
```

Dependency Evidence 재사용은 Application Integration 성공을 의미하지 않는다.

---

## 6. 현재 검증 기준 상태

| 영역 | 상태 | 12 처리 |
| --- | --- | --- |
| Kubernetes / Calico / CoreDNS | Validated | Dependency Evidence 재사용 가능 |
| Namespace / RBAC / Argo CD | Validated | Dependency Evidence 재사용 가능 |
| Redis Runtime / Persistence | Validated | Backend Consumer 별도 검증 |
| Backend → Redis | Not Tested | Planned |
| Backend/Frontend Child Application | Root 편입 완료 | Runtime과 분리 |
| Backend Runtime | `replicas: 0`, `git-pending` | Not Tested |
| Frontend Runtime | `replicas: 0`, `git-pending` | Not Tested |
| A-10 Production Provider Integration | In Progress | Blocker |
| Backend Origin JSON 계약 | GitOps #90 Open | Blocker |
| One-shot Migration 자산 | Implemented / Static+API Validated | 실제 DB Gate Not Tested |
| 실제 Migration Gate | Not Tested | Planned |
| HPA | Not Implemented | Planned |
| Application HTTPRoute | Not Implemented | Planned |
| Application ServiceMonitor | `.pending` | Planned, GitOps #91 |
| Application Metrics | Not Tested | Planned |
| Application Logs | Platform 경로 존재, 실제 App Consumer 미검증 | Planned |

---

## 7. Test Case Traceability Matrix

| Test Case | 상위 검증축/목적 | 09 Gate | 11 연결 | 현재 상태 |
| --- | --- | --- | --- | --- |
| KAI-PRE-01 Dependency Snapshot | 공통 | A~C | 4~6 | Defined |
| KAI-DEL-01 Artifact→GitOps→Argo Traceability | CI/CD Integration | B~D/E | 5 / 8 / 11 / 13 | Blocked |
| KAI-MIG-01 Migration Gate | Runtime/Data Integrity 선행 | C→D | 7 | Not Tested |
| KAI-RUN-01 Backend 1 Replica Provider Runtime | Runtime / M-03 선행 | D | 8 | Blocked |
| KAI-RUN-02 Backend 2 Replica Shared Runtime | M-01 / M-03 | E | 9 | Blocked |
| KAI-HPA-01 HPA / Workload Distribution | M-02 | E | 10 | Not Implemented |
| KAI-FE-01 Frontend Runtime | 통합 | E | 11 | Blocked |
| KAI-RT-01 HTTPRoute / HTTPS / WSS | 통합 / M-03 | F | 12 | Not Implemented |
| KAI-OBS-01 Application Metrics | C-06 / M4 선행 | G | 14 | Blocked |
| KAI-OBS-02 Application Logs | C-06 / M4 선행 | G | 14 | Blocked |
| KAI-E2E-01 Browser M5 First Success | M-01 / M-03 | F~G | 11~14 | Blocked |
| KAI-CON-01 동시성 정확성 | M-01 / P4 | G | 9 | Blocked |
| KAI-PERF-01 Vote 단계 부하 | M-02 / P4 | G | 9~10 | Blocked |
| KAI-REC-01 Backend 장애 복구 | M-03 / P4 | G | 9 / 15 / 17 | Blocked |
| KAI-CD-01 Argo Self-Heal | GitOps | G | 13 | Planned |
| KAI-CD-02 Git Revert Rollback | M-03 / GitOps | G | 13 / 17 | Planned |
| KAI-CFG-01 Config / Secret / CA Consumer 반영 | 공통 | D~G | 16~17 | Planned |
| KAI-DR-01 Restore 후 Application 정상화 | M-04 Cross-role | G | 17 | Planned |

---

# 8. Test Case 상세

## KAI-PRE-01 — Dependency Snapshot

### 목적

Application Runtime 시험 전에 선행 Platform과 작업 대상 Revision을 고정한다.

### 기록

- Node Ready
- Argo Target Revision / Sync / Health
- Backend/Frontend Desired Replica/Image
- Redis Runtime
- Secret Object 및 Key 이름
- CA ConfigMap
- Image Digest
- App/GitOps/Infra Commit SHA

### PASS

- 필요한 Dependency 존재.
- 시험 Revision과 Evidence Metadata 일치.
- Blocker가 있으면 다음 Gate로 진행하지 않음.

### BLOCKED

- Required Dependency 부재
- Revision 불일치
- Secret/CA/Endpoint 부재
- A-10 완료 전 Backend Runtime 강제 시험

---

## KAI-DEL-01 — Application Artifact → GitOps → Argo Traceability

### 목적

Application Source Commit에서 생성된 Image가 Harbor Digest로 식별되고 GitOps와 Argo CD를 거쳐 실제 Pod까지 동일 Artifact로 연결되는지 검증한다.

Jenkins/Harbor 구축 Owner는 Delivery 영역이지만 정태훈 역할은 Application Consumer와 GitOps Desired State 연결을 검증한다.

### Chain

```text
App main Commit
→ Jenkins Run
→ Harbor Tag
→ Harbor Digest
→ GitOps PR
→ GitOps main Revision
→ Argo Sync
→ Deployment
→ Pod ImageID
```

### 기록

- Artifact Target: backend / frontend
- App Commit
- Jenkins Run ID/시간
- Build/Push 시간 — 제공 가능한 경우
- Harbor Tag/Digest
- GitOps PR/Commit/Merge 시각
- Argo Sync/Healthy 시각
- Pod Ready 시각
- Pod ImageID

### PASS

- Commit→Jenkins Run→Digest 연결 가능.
- GitOps가 해당 Digest 고정.
- Argo가 해당 Revision Sync.
- Pod ImageID가 승인 Digest와 일치.
- Runtime에 `latest`/`git-pending` 미사용.

### Evidence

- Jenkins Run
- Harbor Digest
- GitOps PR/Commit
- Argo Sync/Health
- Pod ImageID
- Build/Push/Sync/Ready Timeline

---

## KAI-MIG-01 — 승인형 Migration Gate

### 목적

Backend Runtime 전 실제 DB Revision을 Audit하고 필요한 Action만 승인된 방식으로 수행한 뒤 데이터/복제 상태를 검증한다.

이 Test는 Schema Migration Runtime Gate이며 M-04 DR 자체를 대신하지 않는다.

### 선행조건

공통:

- 승인된 Backend Image Digest
- DB Backup/Restore 준비 상태
- Replication 확인
- Migration Secret
- CA
- Active Migration Job 없음

Mutation(`stamp-baseline`, `upgrade-head`)을 수행하는 경우에만 추가로 **실행 승인 Reference**를 요구한다. Read-only `current` 확인 자체에 Mutation 승인 Reference가 필요하다고 가정하지 않는다.

### 기록

- 실행 전 Alembic Revision
- 선택 Action
- Mutation Approval Reference — 해당 시
- Job 이름/시작/종료
- terminal status
- 실행 후 Revision
- Replication
- MaxScale Read/Write
- 표본 데이터 보존

### PASS

공통:

- Audit 결과와 선택 Action 일치.
- 기대 Revision 확인.
- Replication 정상.
- 표본 데이터 보존.
- Runtime/Migration Credential 경계 유지.

Mutation 수행 시:

- 승인된 Job만 실행.
- Job 성공 종료.
- 동일 Mutation Job 중복 실행 없음.

### FAIL

- Active Mutation 중복
- 승인 없는 Mutation
- Job Failed
- Revision 불일치
- Replication 오류
- 데이터 손실/변형
- Backend가 Migration Credential 소비

### Evidence

- Audit 결과
- Approval Reference — 해당 시
- Rendered Job checksum
- Job status/log
- Revision Before/After
- Replication snapshot
- 표본 데이터 비교

---

## KAI-RUN-01 — Backend 1 Replica Provider First Runtime

### 목적

Production Backend가 Memory Provider 없이 실제 MariaDB·Redis Provider로 1 Replica 기동하고 Provider-aware readiness를 통과하는지 검증한다.

### 선행조건

- A-10 완료
- Backend KAI-DEL-01 PASS
- KAI-MIG-01 PASS
- GitOps #90 Origin JSON 계약 완료
- Runtime Secret / CA / Redis 준비
- Digest Pinning
- `replicas: 1`

### 기록

- Argo Sync/Health
- Deployment desired/ready
- Pod/Image Digest
- Startup/Liveness/Readiness
- MaxScale TLS
- Identity/Game DB
- Redis
- Provider readiness
- 최소 Backend First Runtime 결과

### PASS

- Backend Pod 1 Ready.
- 승인 Digest 일치.
- Startup/Live/Ready 정상.
- Identity/Game DB TLS 연결 성공.
- Redis Consumer 성공.
- Migration Credential 미소비.
- production 경로에 Memory Provider fallback 없음.

---

## KAI-RUN-02 — Backend 2 Replica Shared Runtime

### 목적

2 Replica에서 권위 상태가 Pod-local Memory로 분리되지 않고 Redis/DB 공유 상태를 통해 수렴하는지 확인한다.

### 선행조건

- KAI-RUN-01 PASS
- A-10 Shared State/Realtime/Presence/Runner 완료
- 동일 Image Digest

### 기록

- Pod 2 Ready
- Instance ID
- Worker 배치
- Replica 간 동일 Room/Game 결과
- Redis shared state
- Pub/Sub
- Reconnect
- Runner Lease/중복 실행

### PASS

- 두 Replica 권위 상태 수렴.
- Process-local Memory가 권위 상태 아님.
- Pub/Sub 누락/재연결 시 Snapshot으로 수렴.
- Runner 중복 마감 없음.

---

## KAI-HPA-01 — HPA / Workload Distribution

### 상태

현재 **Not Implemented / Not Tested**.

### 선행조건

- KAI-RUN-01 PASS
- KAI-RUN-02 PASS
- Resource Request/Limit
- Resource Metrics
- HPA ↔ Argo Replica Ownership 계약
- HPA Manifest Merge

### 측정값

```text
부하 단계
HPA target/current metric
Desired / Current Replica
Scale-out/in 시작·완료 시각
p95/p99
Error rate
Pod Ready
Argo Diff/Sync
```

### PASS

- HPA Metrics 정상.
- 설정 범위 안에서 Replica 변화.
- Scale 중 가용성 유지.
- Argo와 `spec.replicas` 경쟁 없음.
- M-01 정확성 오류 없음.

임계값 자체의 적절성은 단계 부하 결과로 판단한다.

---

## KAI-FE-01 — Frontend Runtime

### 선행조건

- 09 Critical Path의 Backend Scale-out / HPA·Workload Distribution 단계 판정 완료
- Frontend KAI-DEL-01 PASS
- Container Runtime Smoke
- `apps-frontend` 정상

HPA가 일정상 Deferred되면 PASS로 가장하지 않고 별도 Go/No-Go 근거를 남긴 뒤 Frontend 진행 여부를 결정한다.

### 기록

- GitOps Revision
- Image Digest
- Deployment desired/ready
- Pod/Service
- SPA fallback
- Container health

### PASS

- 승인 Digest 실행.
- Pod/Service 정상.
- SPA 기본 경로 정상.
- Browser E2E와 구분.

---

## KAI-RT-01 — Application HTTPRoute / HTTPS / WSS

### 상태

현재 **Not Implemented / Not Tested**.

### 경로

```text
/       → frontend:8080
/api/v1 → backend:8000
/ws/v1  → backend:8000
```

### 선행조건

- Backend Runtime
- Frontend Runtime
- Gateway Platform
- HTTPRoute Merge

### 기록

- HTTPRoute `Accepted` / `ResolvedRefs`
- HTTPS
- API
- WSS handshake/message
- Forwarded Header
- Cookie/CORS
- DELETE Body
- 내부 Endpoint 비공개

### PASS

- Route Condition 정상.
- HTTPS Frontend 성공.
- `/api/v1` Backend 전달.
- `/ws/v1` WSS 성공.
- Browser 조건의 CORS/Cookie/Forwarded Header 정상.
- 내부 Health/Metric 의도치 않은 외부 공개 없음.

---

## KAI-OBS-01 — Application Metrics

### 상태

ServiceMonitor는 `.pending`, GitOps #91 추적.

### 선행조건

- Backend Runtime Running
- `/metrics` 제공
- Backend Service `metadata.labels` 계약
- ServiceMonitor selector 정합
- `.pending` 제거/Merge

### 기록

- ServiceMonitor
- Service `metadata.labels`
- ServiceMonitor selector
- Prometheus Target
- scrape error
- metric sample

### PASS

- Target `UP`.
- ServiceMonitor가 의도한 Service 선택.
- 실제 Application Metric 조회.
- Platform Running과 Application Metrics 성공 분리.

---

## KAI-OBS-02 — Application Logs

### 목적

Backend/Frontend stdout/stderr가 Alloy→Loki로 수집되고 Pod Identity를 추적할 수 있는지 확인한다.

### Platform label

```text
namespace
pod
container
node_name
```

### 기록

- Pod/Container/Node
- 기준 Log Event 시각
- Loki 조회
- label 일치
- 수집 지연

### PASS

- Application Pod Log가 Loki에서 조회됨.
- label이 실제 Pod와 일치.
- 민감정보 Raw Log 노출 없음.
- Platform Running과 실제 App Log 수집 성공 분리.

### Cross-role Alert

Alertmanager Rule·firing/resolved·E-mail 통보 검증은 Delivery/Observability 담당이 주 Owner다. 최종 MVP Acceptance에서는 해당 Evidence를 같은 장애/측정 Run과 연결한다.

---

## KAI-E2E-01 — Browser M5 First Success

### 목적

M3 Runtime과 M4 Delivery/Observability 선행조건 이후 실제 공개 Hostname에서 Frontend→Gateway→Backend→DB/Redis가 연결되는 M5 First Success를 증명한다.

### 선행조건

- Backend Runtime PASS
- Frontend Runtime PASS
- HTTPRoute/HTTPS/WSS PASS
- 실제 Provider 사용
- Delivery Artifact Traceability 확보
- Application Metrics/Logs 또는 현재 M4에서 요구하는 Delivery/Observability 선행 Evidence 확보

### 사용자 흐름

실행 시점의 최신 `MVP_IMPLEMENTATION_BASELINE`과 A-10/M5 정의를 기준으로 **구현 완료된 M5 핵심 흐름**을 사용한다. 문서가 미구현 기능을 임의로 First Success 필수조건으로 추가하지 않는다.

대표 흐름:

```text
Frontend 접속
→ 회원/세션
→ Lobby/Room
→ 팀/Ready
→ Game/Vote
→ Move
→ Realtime Update
→ 결과 확인
```

### PASS

- 실제 Browser HTTPS/WSS.
- Same-Origin 계약 유지.
- Backend/Redis/DB 실제 Provider 사용.
- 핵심 상태 일관성 유지.
- Fake/Memory E2E와 구분.

### Evidence

- Browser timestamp/screenshot
- Network trace 요약
- Backend Instance ID
- Application logs
- Result/Move state
- 관련 Metric/Log Evidence

---

## KAI-CON-01 — M-01 동시성 정확성 / P4

### 목적

Vote 등록·변경·삭제·마감·재시도와 stale 요청이 겹쳐도 Move/Result/Rating이 중복·유실되지 않는지 검증한다.

### 측정값

```text
정상 Turn 확정 Move 수
Pass Turn 확정 Move 수
중복 GameResult
중복 Rating
stale 요청 상태 변경
idempotency 중복 반영
```

### PASS

```text
정상 Turn Move = 1
Pass Turn Move = 0
중복 GameResult = 0
중복 Rating = 0
stale 잘못된 상태 변경 = 0
```

동일 idempotency request의 권위 반영도 1회여야 한다.

Backend-only 단계에서 사전 결함 탐지용으로 실행할 수 있지만 **최종 P4 판정은 M5 First Success 이후 실제 MVP Runtime Revision으로 다시 실행**한다.

---

## KAI-PERF-01 — M-02 Vote 단계 부하 / P4

### 측정값

```text
Vote events/s
p50 / p95 / p99
Error rate
Timeout
CPU / Memory
Replica
Redis/DB 진단
WebSocket connection
Active Room/Game
```

### 부하 방식

```text
낮은 Baseline
→ 단계 증가
→ 안정 구간 측정
→ p95/p99·오류·자원 관찰
→ 급격한 악화 지점 확인
→ 병목 추적
```

### PASS/판정

- 단계별 처리량·p95/p99·오류율 측정 완료.
- M-01 오류 없음.
- 병목 단계와 원인 진단 가능.
- 근거 없는 고정 목표값/개선률을 만들지 않음.

현재 MVP에서 ANALYSIS Runtime은 제외되어 있으므로 AI 분석 조건은 현재 PASS 필수조건이 아니다.

Backend-only 사전 부하는 가능하지만 **P4 Baseline은 M5 First Success 이후 실제 MVP Runtime Revision에서 측정**한다.

---

## KAI-REC-01 — M-03 Backend 장애 복구 / P4

### 선행조건

- KAI-RUN-02 PASS
- Reconnect/Snapshot 복구 구현
- 장애 범위/중단조건 합의

### 측정값

```text
장애 시각
첫 실패 요청
재연결 성공
서비스 복구
상태 복구 완료
실패 요청 수
잘못된 사용자 패배 판정
Room/Game/Turn/Board Before/After
```

### PASS

- 단일 Backend 장애가 몰수패/공동패배로 처리되지 않음.
- 잘못된 사용자 패배 = 0.
- 권위 상태 복원.
- 복구시간 측정 가능.

Backend-only 사전 장애 시험은 가능하지만 **최종 P4 장애/복구 Evidence는 M5 First Success 이후 실제 MVP Runtime Revision으로 다시 확보**한다.

---

## KAI-CD-01 — Argo CD Self-Heal

### 안전 원칙

- Production 데이터 미변경.
- 복구 가능한 비영속/무해 필드 사용.
- 정상 Git Revision 고정.
- 11에서 허용한 명시적 검증용 Live Drift로만 수행.

### 측정

```text
Drift 주입
OutOfSync 감지
Self-Heal 시작/완료
최종 Sync/Health
Live Before/After
```

### PASS

- Drift 감지.
- Self-Heal 후 Git Desired State와 일치.
- 불필요한 가용성 영향 없음.

---

## KAI-CD-02 — Git Revert Rollback

DB Schema Migration은 대상이 아니다.

### 측정

```text
문제 Revision 인지
Revert PR 생성/승인/Merge
Argo Sync
Pod Ready
서비스 정상화
총 Recovery 시간
수동 단계
```

### PASS

- Git main Known-Good로 복귀.
- Argo Sync.
- Known-Good Runtime 복구.
- 장기 Cluster patch 잔존 없음.

---

## KAI-CFG-01 — Config / Secret / CA Consumer 반영

### 핵심 경계

- Secret env는 기존 프로세스에 자동 재주입되지 않음.
- ConfigMap `envFrom`도 기존 프로세스에 자동 재주입되지 않음.
- `subPath` CA는 ConfigMap 변경만으로 기존 Container 파일 갱신 안 됨.

### 안전 원칙

이 Test만을 위해 운영 Credential이나 CA를 임의 회전하지 않는다. 실제 승인된 변경 이벤트가 있으면 해당 Run에서 검증하거나, 비밀이 아닌 Config 변경으로 Pod 반영 경계를 검증한다.

### 기록

- 변경 전/후 Pod UID
- Git/Infra Revision
- Rollout 여부
- non-secret Config 또는 CA fingerprint
- Readiness

### PASS

- 필요한 경우 새 Pod 생성.
- 새 Pod가 최신 Desired State 소비.
- CA 변경 시 기대 fingerprint.
- Git 기록 없는 `kubectl rollout restart`만으로 종료하지 않음.

---

## KAI-DR-01 — Restore 후 Application 정상화 Cross-role

M-04 Backup/Restore·RTO/RPO는 Data/Storage Recovery 담당이 주 Owner다.

본 역할 확인:

- 복원 DB Endpoint 연결
- Redis/DB 상태 경계
- Member/MemberStats/GameResult/Move/RatingHistory 표본 조회
- Readiness
- Browser/API 핵심 흐름

### PASS

- Restore Evidence와 Application Consumer Evidence가 같은 Run/Revision으로 연결.
- 표본 무결성 PASS.
- Application 정상화.

다른 역할의 RTO/RPO를 본 문서에서 재정의하지 않는다.

---

# 9. 검증 Phase와 실행 순서

## Phase 0 — Dependency / Artifact

```text
KAI-PRE-01
→ KAI-DEL-01 Backend Artifact
→ KAI-MIG-01
```

## Phase 1 — Backend First Runtime

```text
KAI-RUN-01 Backend 1 Replica
→ Backend-only 기능/정확성 사전 검증 가능
```

## Phase 2 — Scale / HPA

```text
KAI-RUN-02 Backend 2 Replica
→ KAI-HPA-01
```

HPA가 Deferred되면 PASS로 기록하지 않고 Go/No-Go 근거와 상태를 남긴다.

## Phase 3 — Frontend / External / Delivery-Observability

```text
KAI-DEL-01 Frontend Artifact
→ KAI-FE-01
→ KAI-RT-01
→ KAI-OBS-01 / KAI-OBS-02
→ Delivery/Observability Cross-role prerequisite 확인
```

## Phase 4 — M5 First Success

```text
KAI-E2E-01
```

09 기준 M3 Runtime과 M4 Delivery/Observability 선행조건을 충족한 뒤 수행한다.

## Phase 5 — MVP P4

M5 First Success가 성립한 **동일한 MVP Runtime 계열**에서 대표 부하·동시성·장애/복구를 최종 수행한다.

```text
KAI-CON-01
KAI-PERF-01
KAI-REC-01
KAI-CD-01 / KAI-CD-02
KAI-CFG-01 — 실제 변경 시나리오가 있을 때
Cross-role KAI-DR-01
```

Backend-only 단계에서 동일 유형 Test를 사전 수행한 결과는 결함 탐지 Evidence로 보존할 수 있으나 P4 최종 결과를 대신하지 않는다.

### 절대 Gate

```text
A-10 미완료 → Backend Runtime 금지
Backend 1 Replica 미검증 → 2 Replica/HPA 금지
2 Replica Shared Runtime 미검증 → M-01/M-03 최종 판정 금지
Frontend/Backend Service 미준비 → HTTPRoute E2E 금지
Metrics Endpoint/Service metadata label 미정 → ServiceMonitor 활성화 금지
M3/M4 선행조건 미충족 → M5 First Success 최종 판정 금지
M5 미성립 → P4 최종 판정 금지
```

---

# 10. Before → Change → After

## 10.1 Replica / HPA

```text
Before: 검증된 Static Replica
Change: Scale-out 또는 HPA
After: 동일 부하
```

비교:

```text
Vote events/s
p95/p99
Error
Resource
Replica
M-01 오류
```

## 10.2 장애 복구

```text
Before: 정상 Snapshot
Change: 통제 Backend 장애
After: 재연결/복구 Snapshot
```

비교:

```text
상태 일관성
복구시간
실패 요청
잘못된 패배 판정
```

## 10.3 GitOps Rollback

```text
Before: Known-Good
Change: 검증 가능한 Revision/Drift
After: Revert + Argo Sync
```

비교:

```text
Recovery time
Manual steps
Sync/Health
Pod Ready
서비스 정상화
```

## 10.4 Artifact Delivery

```text
App Commit
→ Build / Push
→ GitOps Update
→ Argo Sync
→ Pod Ready
```

기록:

```text
Build 시간
Push 시간
GitOps PR/Merge 시간
Argo Sync 시간
Pod Ready 시간
Commit → Digest → Pod Traceability
```

---

# 11. 실패/중단 판정

즉시 중단:

- 데이터 손실 위험
- 예상 외 DB Revision
- Replication 오류
- Secret/Token/Password 노출
- 잘못된 GameResult/Rating 반영
- M-01 정확성 오류
- Backend 장애가 사용자 패배로 잘못 확정
- HPA/Argo Replica ownership 지속 경쟁
- 의도한 Revision과 다른 Image/Manifest 사용

```text
Run = FAILED 또는 BLOCKED
→ 원인 기록
→ 영향 범위 고정
→ 필요 시 11 Rollback/Recovery
→ 새 Run ID 재실행
```

---

# 12. 결과 기록 Template

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
- ...

Measured:
- ...

PASS/FAIL Basis:
- ...

Evidence:
- Issue/PR:
- Command/Runbook:
- Log:
- Metric/Export:
- Screenshot:
- Snapshot/Checksum:

Failure/Exception:
- ...

Manual Steps:
- ...

Cleanup/Rollback:
- ...

Follow-up:
- ...
```

---

# 13. MVP Acceptance 연결

Gate G는 본 역할의 Test만으로 완료되지 않는다.

```text
Kubernetes/Application Integration Evidence
+ Data/Storage Recovery Evidence
+ Delivery/Observability Evidence
+ Network/External Infra Evidence
+ Browser/E2E Evidence
```

본 역할 Acceptance 핵심:

- Commit→Digest→GitOps→Argo→Pod Traceability
- Migration Gate Evidence
- Backend 1 Replica Provider Runtime PASS
- Backend 2 Replica Shared Runtime PASS
- Scale/HPA 상태 명확화
- Frontend Runtime PASS
- HTTPRoute/HTTPS/WSS PASS
- Application Metrics/Logs Consumer Evidence
- Browser M5 First Success PASS
- M-01 P4 PASS
- M-02 P4 Baseline 확보
- M-03 P4 PASS + 복구시간 측정
- GitOps Self-Heal/Rollback Evidence
- HPA 미구현/Deferred 시 이를 PASS로 가장하지 않고 근거 유지

Delivery/Observability 담당의 Alert firing/resolved/E-mail Evidence와 Data/Storage Recovery의 DR Evidence는 동일 Acceptance Run과 연결한다.

---

# 14. 문서 완료 기준

- 02 M-01~M-05 Traceability.
- 07 Commit→Digest→Healthy 및 Metric/Log/Evidence 원칙과 정합.
- 09 Critical Path, M5 First Success, P4 순서 유지.
- 11 실행절차 반복 없이 Test/Measurement/Evidence 연결.
- 각 핵심 Test에 목적·측정값·PASS/FAIL·Evidence 존재.
- 미실행 항목을 결과처럼 작성하지 않음.
- 성능 목표/HPA 임계/RTO/RPO 임의 생성 금지.
- ANALYSIS Runtime을 현재 MVP PASS 필수조건으로 만들지 않음.
- Dependency Evidence와 Consumer 검증 구분.
- Cross-role Owner 경계 명확.
- Secret/Token/Private Key Evidence 금지.
- 실패 Run 보존, 재실행은 새 Run ID.
- 실제 Repository/Runtime과 재대조 후 추가 필수 보완사항이 없을 때 종료.
