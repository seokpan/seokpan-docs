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
Runtime Registry Pull / imagePullSecret Consumer
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
Registry Pull Capability ≠ Actual Workload Pull
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
- Infra #180/PR #185의 Runtime pull-only Robot·`application/harbor-pull-secret` 공급 및 **임시 Pod Digest Pull** Evidence — 해당 Revision이 확정된 경우

다만 Registry Capability Evidence를 실제 Backend/Frontend Deployment Pull 성공으로 대신하지 않는다.

재검증 조건:

```text
관련 Manifest / Playbook / Image 변경
Version 변경
Secret / CA 변경
Registry Credential 변경
Cluster 재구축
장애/복구 후
실제 Consumer 연결 최초 수행
Evidence 대상 Revision과 현재 Revision 불일치
```

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
| Runtime Pull-only Robot / Vault / `application/harbor-pull-secret` | Infra PR #185 Open, 임시 Pod Digest Pull PASS | Partial / Merge 대기 |
| Backend/Frontend Deployment `imagePullSecrets` | GitOps #57 후속 | Not Yet Wired / Not Tested |
| Actual Backend/Frontend Workload Pull | GitOps #57 후속 | Not Tested |
| Infra #180 | Open | Actual Workload 소비 확인 후 종료 |
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
| KAI-DEL-01 Artifact/Registry/GitOps/Argo Traceability | CI/CD Integration | B~D/E | 5 / 8 / 11 / 13 | Partial / Blocked |
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
- Registry Pull Secret Metadata
- Image Digest
- App/GitOps/Infra Commit SHA

### PASS

- 필요한 Dependency 존재.
- 시험 Revision과 Evidence Metadata 일치.
- Blocker가 있으면 다음 Gate로 진행하지 않음.

---

## KAI-DEL-01 — Artifact / Registry Pull / GitOps / Argo Traceability

### 목적

Application Source Commit에서 생성된 Image가 Harbor Digest로 식별되고, Runtime 전용 최소권한 Registry Credential을 통해 Kubernetes가 해당 Digest를 Pull하며, GitOps/Argo CD를 거쳐 **실제 Backend/Frontend Workload**까지 같은 Artifact로 연결되는지 검증한다.

Jenkins/Harbor/Robot 공급 자체는 Delivery/Infra 영역과 Cross-role이며, 정태훈 역할은 Application Workload Consumer 및 GitOps Desired State 연결을 검증한다.

### 8.2.1 Traceability Chain

```text
App main Commit
→ Jenkins Run
→ Harbor Tag / Digest
→ Runtime Pull-only Robot
→ application/harbor-pull-secret
→ GitOps imagePullSecrets + Digest
→ GitOps PR / Merge
→ Argo Sync
→ Actual Backend/Frontend Pod Pull
→ Pod ImageID
→ Workload Running / Ready
```

### 8.2.2 Capability와 Consumer를 구분한다

**Registry Pull Capability**:

```text
pull-only Robot
→ Vault 등록
→ application/harbor-pull-secret
→ 임시 Pod의 검증된 Digest Pull
```

Infra PR #185에서 이 범위의 Evidence가 준비되어 있다. 단 PR이 Merge되기 전에는 `main`의 확정 Capability로 승격하지 않는다.

**Actual Workload Consumption**:

```text
Backend/Frontend Deployment
→ imagePullSecrets: harbor-pull-secret
→ 검증된 Digest
→ Argo Sync
→ 실제 Application Pod Pull / Start / Ready
```

이 범위는 GitOps #57 후속이며 아직 완료로 판정하지 않는다.

### 8.2.3 기록

- Artifact Target: backend / frontend
- App Commit
- Jenkins Run ID/시간
- Harbor Tag/Digest
- Infra PR/Commit
- Runtime Robot 이름/권한 Metadata — Secret 제외
- `application/harbor-pull-secret` 이름/Namespace/Type — Data 제외
- 임시 Pod Digest Pull Evidence — Capability
- GitOps PR/Commit/Merge
- Deployment `imagePullSecrets`
- Deployment Image Digest
- Argo Sync/Healthy 시각
- Pod Pull/Start/Ready 시각
- Pod ImageID

### PASS

전체 KAI-DEL-01 PASS는 다음을 모두 만족해야 한다.

- App Commit→Jenkins→Harbor Digest 연결 가능.
- Runtime Robot 권한이 pull-only.
- `application/harbor-pull-secret`이 Runtime 전용 Credential을 사용.
- Backend/Frontend Deployment가 Secret을 명시적으로 소비.
- GitOps가 검증된 Digest를 고정.
- Argo가 해당 Revision Sync.
- 실제 Workload가 Private Image Pull 성공.
- Pod ImageID가 승인 Digest와 일치.
- Runtime에 `latest`/`git-pending` 미사용.

### Partial

Infra #180/PR #185의 임시 Pod Pull만 PASS한 경우:

```text
Registry Capability = Validated/Review Pending
Actual Workload Consumption = Not Tested
KAI-DEL-01 전체 = Partial / Blocked
```

### FAIL

- Runtime에 CI Push 권한 Credential 사용
- Deployment `imagePullSecrets` 누락
- `ErrImagePull` / `ImagePullBackOff`
- GitOps Digest와 Pod ImageID 불일치
- Commit→Digest 연결 불가

### Evidence

- Jenkins Run
- Harbor Digest
- Robot 권한 Metadata
- Secret Metadata
- Infra #180 / PR #185
- GitOps #57 PR/Commit
- Argo Sync/Health
- Pod Event / ImageID
- Pull/Start/Ready Timeline

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

Mutation(`stamp-baseline`, `upgrade-head`)을 수행하는 경우에만 실행 승인 Reference를 요구한다. Read-only `current` 확인 자체에 Mutation 승인 Reference가 필요하다고 가정하지 않는다.

### 기록

- 실행 전/후 Alembic Revision
- 선택 Action
- Mutation Approval Reference — 해당 시
- Job 이름/상태/시간
- Replication
- MaxScale Read/Write
- 표본 데이터 보존

### PASS

- Audit와 Action 일치.
- 기대 Revision.
- Replication 정상.
- 표본 데이터 보존.
- Runtime/Migration Credential 경계 유지.
- Mutation 시 승인된 Job만 1회 수행.

---

## KAI-RUN-01 — Backend 1 Replica Provider First Runtime

### 목적

Production Backend가 실제 MariaDB·Redis Provider로 1 Replica 기동하고 Provider-aware readiness를 통과하는지 검증한다.

### 선행조건

- A-10 완료
- Backend KAI-DEL-01의 **실제 Backend Workload Pull** PASS
- KAI-MIG-01 PASS
- GitOps #90 Origin JSON 계약 완료
- Runtime Secret / CA / Redis
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

### PASS

- Backend Pod 1 Ready.
- 승인 Digest 일치.
- DB TLS/Redis Consumer 성공.
- Migration Credential 미소비.
- production 경로에 Memory Provider fallback 없음.

---

## KAI-RUN-02 — Backend 2 Replica Shared Runtime

### 선행조건

- KAI-RUN-01 PASS
- A-10 Shared State/Realtime/Presence/Runner 완료
- 동일 Image Digest

### PASS

- Pod 2 Ready.
- 권위 상태 Replica 간 수렴.
- Process-local Memory가 권위 상태 아님.
- Pub/Sub 누락/재연결 시 Snapshot 수렴.
- Runner 중복 마감 없음.

---

## KAI-HPA-01 — HPA / Workload Distribution

### 상태

현재 **Not Implemented / Not Tested**.

### 선행조건

- KAI-RUN-02 PASS
- Resource Request/Limit
- Resource Metrics
- HPA ↔ Argo Replica Ownership 계약
- HPA Manifest Merge

### PASS

- Metrics 정상.
- 설정 범위 안 Replica 변화.
- Scale 중 가용성 유지.
- Argo와 `spec.replicas` 경쟁 없음.
- M-01 오류 없음.

---

## KAI-FE-01 — Frontend Runtime

### 선행조건

- Backend Scale-out / HPA·Workload Distribution 단계 판정
- Frontend KAI-DEL-01의 **실제 Frontend Workload Pull** PASS
- Container Smoke
- `apps-frontend` 정상

HPA가 Deferred되면 PASS로 가장하지 않고 Go/No-Go 근거를 남긴다.

### PASS

- 승인 Digest 실행.
- Frontend Pod/Service 정상.
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

### PASS

- HTTPRoute `Accepted` / `ResolvedRefs` 정상.
- HTTPS Frontend 성공.
- `/api/v1` Backend 전달.
- `/ws/v1` WSS 성공.
- CORS/Cookie/Forwarded Header 정상.
- 내부 Health/Metric 의도치 않은 외부 공개 없음.

---

## KAI-OBS-01 — Application Metrics

### 상태

ServiceMonitor는 `.pending`, GitOps #91 추적.

### PASS

- Backend Runtime이 `/metrics` 제공.
- Service `metadata.labels`와 ServiceMonitor selector 정합.
- Prometheus Target `UP`.
- 실제 Application Metric 조회.

---

## KAI-OBS-02 — Application Logs

### 목적

Backend/Frontend stdout/stderr가 Alloy→Loki로 수집되고 Pod Identity를 추적할 수 있는지 검증한다.

### PASS

- Application Pod Log Loki 조회.
- `namespace/pod/container/node_name` 실제 Pod와 일치.
- 민감정보 Raw Log 노출 없음.

Alert firing/resolved·E-mail 통보는 Delivery/Observability 담당이 주 Owner이며 최종 Acceptance에서 동일 Run과 연결한다.

---

## KAI-E2E-01 — Browser M5 First Success

### 목적

M3 Runtime과 M4 Delivery/Observability 선행조건 이후 실제 공개 Hostname에서 Frontend→Gateway→Backend→DB/Redis가 연결되는 M5 First Success를 증명한다.

### 선행조건

- Backend/Frontend Actual Workload Pull PASS
- Backend Runtime PASS
- Frontend Runtime PASS
- HTTPRoute/HTTPS/WSS PASS
- 실제 Provider
- Artifact Traceability
- Application Metrics/Logs 또는 현재 M4 선행 Evidence

### 사용자 흐름

실행 시점 최신 `MVP_IMPLEMENTATION_BASELINE`과 A-10/M5 정의를 기준으로 구현 완료된 M5 핵심 흐름을 사용한다.

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

- 팀원 PC의 실제 URL에서 HTTPS/WSS 사용.
- Same-Origin 계약 유지.
- Backend/Redis/DB 실제 Provider.
- 실제 데이터 UX/UI가 기능 계약과 정합.
- Fake/Memory E2E와 구분.

---

## KAI-CON-01 — M-01 동시성 정확성 / P4

### PASS

```text
정상 Turn Move = 1
Pass Turn Move = 0
중복 GameResult = 0
중복 Rating = 0
stale 잘못된 상태 변경 = 0
```

동일 idempotency request 권위 반영도 1회.

Backend-only 사전 시험은 가능하지만 최종 P4는 M5 이후 실제 MVP Runtime Revision에서 재실행한다.

---

## KAI-PERF-01 — M-02 Vote 단계 부하 / P4

### 측정

```text
Vote events/s
p50/p95/p99
Error rate
Timeout
CPU/Memory
Replica
Redis/DB 진단
WebSocket connection
Active Room/Game
```

### PASS/판정

- 단계별 지표 측정 완료.
- M-01 오류 없음.
- 병목 단계/원인 식별 가능.
- 근거 없는 목표/개선률 미생성.

ANALYSIS Runtime은 현재 PASS 필수조건이 아니다.

---

## KAI-REC-01 — M-03 Backend 장애 복구 / P4

### PASS

- 단일 Backend 장애가 몰수패/공동패배로 처리되지 않음.
- 잘못된 사용자 패배 = 0.
- 권위 상태 복원.
- 복구시간 측정 가능.

최종 P4 Evidence는 M5 이후 실제 MVP Runtime Revision으로 확보한다.

---

## KAI-CD-01 — Argo CD Self-Heal

### 안전 원칙

- Production 데이터 미변경.
- 복구 가능한 비영속/무해 필드 사용.
- 정상 Git Revision 고정.

### PASS

- Live Drift 감지.
- Self-Heal 후 Git Desired State와 일치.
- 불필요한 가용성 영향 없음.

---

## KAI-CD-02 — Git Revert Rollback

DB Schema Migration은 대상이 아니다.

### PASS

- Git main Known-Good 복귀.
- Argo Sync.
- Known-Good Runtime 복구.
- 장기 Cluster patch 잔존 없음.
- Recovery time / Manual steps 측정.

---

## KAI-CFG-01 — Config / Secret / CA Consumer 반영

### 안전 원칙

이 Test만을 위해 운영 Credential이나 CA를 임의 회전하지 않는다. 실제 승인된 변경 이벤트가 있으면 해당 Run에서 검증하거나 비밀이 아닌 Config 변경으로 Pod 반영 경계를 검증한다.

### PASS

- 필요한 경우 새 Pod 생성.
- 새 Pod가 최신 Desired State 소비.
- CA 변경 시 기대 fingerprint.
- Git 기록 없는 `kubectl rollout restart`만으로 종료하지 않음.

---

## KAI-DR-01 — Restore 후 Application 정상화 Cross-role

M-04 Backup/Restore·RTO/RPO는 Data/Storage Recovery 담당이 주 Owner다.

### PASS

- Restore Evidence와 Application Consumer Evidence가 같은 Run/Revision으로 연결.
- 표본 데이터 무결성 PASS.
- Application 정상화.

---

# 9. 검증 Phase와 실행 순서

## Phase 0 — Runtime Registry / Artifact 선행

현재 최신 선행 흐름:

```text
Infra PR #185 리뷰/승인/Merge
→ Runtime pull-only Robot/Vault/Secret을 main 기준 Capability로 확정
→ GitOps #57에서 Backend/Frontend imagePullSecrets + 검증 Digest 반영
→ Argo CD Sync
→ 실제 Backend/Frontend Workload Pull·기동 확인
→ Infra #180 종료
```

이 단계에서 임시 Pod Pull 성공을 실제 Workload Pull 성공으로 대체하지 않는다.

## Phase 1 — Backend First Runtime

```text
A-10 Provider 구현
→ KAI-MIG-01
→ KAI-RUN-01 Backend 1 Replica
```

실제 일정에서는 GitOps #57의 Digest/imagePullSecrets 준비를 A-10과 병렬로 준비할 수 있지만, Provider First Runtime PASS는 Migration과 실제 Backend Consumer 연결까지 필요하다.

## Phase 2 — Scale / HPA

```text
KAI-RUN-02 Backend 2 Replica
→ KAI-HPA-01
```

HPA가 Deferred되면 PASS로 기록하지 않고 근거를 남긴다.

## Phase 3 — Frontend / Gateway / Public Access

```text
KAI-FE-01 Frontend Runtime
→ KAI-RT-01 HTTPRoute / HTTPS / WSS
→ DNS / 공개 TLS 통합 확인
→ KAI-OBS-01 / KAI-OBS-02
```

Gateway Platform의 기존 HTTPS/TLS PASS를 실제 Application Route 성공으로 대신하지 않는다.

## Phase 4 — M5 First Success

```text
팀원 PC에서 실제 URL 접속
→ KAI-E2E-01
→ 실제 데이터 UX/UI 검증
```

M3 Runtime과 M4 Delivery/Observability 선행조건 이후 수행한다.

## Phase 5 — MVP P4

M5 First Success가 성립한 동일 MVP Runtime 계열에서:

```text
KAI-CON-01
KAI-PERF-01
KAI-REC-01
KAI-CD-01 / KAI-CD-02
KAI-CFG-01 — 실제 변경 시나리오가 있을 때
Cross-role KAI-DR-01
```

Backend-only 사전 시험은 결함 탐지 Evidence로 보존할 수 있으나 P4 최종 결과를 대신하지 않는다.

### 절대 Gate

```text
Infra PR #185 미확정 → Registry Capability를 main 완료로 표시 금지
Deployment imagePullSecrets 미반영 → Actual Workload Pull PASS 금지
Actual Workload Pull 미검증 → Infra #180 완료 금지
A-10 미완료 → Provider Runtime PASS 금지
Backend 1 Replica 미검증 → 2 Replica/HPA 금지
2 Replica Shared Runtime 미검증 → M-01/M-03 최종 판정 금지
Frontend/Backend Service 미준비 → HTTPRoute E2E 금지
Metrics Endpoint/Service metadata label 미정 → ServiceMonitor 활성화 금지
M3/M4 선행조건 미충족 → M5 최종 판정 금지
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

비교: Vote events/s, p95/p99, Error, Resource, Replica, M-01 오류.

## 10.2 장애 복구

```text
Before: 정상 Snapshot
Change: 통제 Backend 장애
After: 재연결/복구 Snapshot
```

비교: 상태 일관성, 복구시간, 실패 요청, 잘못된 패배 판정.

## 10.3 GitOps Rollback

```text
Before: Known-Good
Change: 검증 가능한 Revision/Drift
After: Revert + Argo Sync
```

비교: Recovery time, Manual steps, Sync/Health, Pod Ready, 서비스 정상화.

## 10.4 Artifact / Registry Delivery

```text
App Commit
→ Build / Push
→ Runtime Pull Credential
→ GitOps Digest/imagePullSecrets
→ Argo Sync
→ Actual Pod Pull / Ready
```

기록: Build/Push 시간, PR/Merge, Argo Sync, Pull, Pod Ready, Commit→Digest→Pod Traceability.

---

# 11. 실패/중단 판정

즉시 중단:

- 데이터 손실 위험
- 예상 외 DB Revision
- Replication 오류
- Secret/Token/Password 노출
- Runtime에 CI Push Credential 사용
- 의도하지 않은 Image Digest
- 잘못된 GameResult/Rating
- M-01 오류
- Backend 장애가 사용자 패배로 잘못 확정
- HPA/Argo Replica ownership 지속 경쟁

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

- Commit→Digest→Runtime Pull Secret→GitOps→Argo→Actual Pod Traceability
- Migration Gate Evidence
- Backend 1 Replica Provider Runtime PASS
- Backend 2 Replica Shared Runtime PASS
- Scale/HPA 상태 명확화
- Frontend Runtime PASS
- HTTPRoute/HTTPS/WSS/DNS/Public TLS PASS
- Application Metrics/Logs Consumer Evidence
- Browser M5 First Success 및 실제 데이터 UX/UI PASS
- M-01 P4 PASS
- M-02 P4 Baseline
- M-03 P4 PASS + 복구시간
- GitOps Self-Heal/Rollback Evidence
- HPA 미구현/Deferred 시 상태와 근거 유지

Delivery/Observability의 Alert Evidence와 Data/Storage Recovery의 DR Evidence를 동일 Acceptance Run과 연결한다.

---

# 14. 문서 완료 기준

- 02 M-01~M-05 Traceability.
- 07 Commit→Digest→Healthy 및 Metric/Log/Evidence 원칙과 정합.
- 09 Critical Path, M5 First Success, P4 순서 유지.
- 11 실행절차 반복 없이 Test/Measurement/Evidence 연결.
- Infra #180/PR #185와 GitOps #57의 Runtime Pull 경계를 실제 상태대로 반영.
- Registry Capability와 Actual Workload Consumer 검증 구분.
- 각 핵심 Test에 목적·측정값·PASS/FAIL·Evidence 존재.
- 미실행 항목을 결과처럼 작성하지 않음.
- 성능 목표/HPA 임계/RTO/RPO 임의 생성 금지.
- ANALYSIS Runtime을 현재 MVP PASS 필수조건으로 만들지 않음.
- Cross-role Owner 경계 명확.
- Secret/Token/Private Key Evidence 금지.
- 실패 Run 보존, 재실행은 새 Run ID.
- 실제 Repository/Runtime과 재대조 후 추가 필수 보완사항이 없을 때 종료.
