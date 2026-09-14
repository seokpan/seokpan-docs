# MVP 검증·측정 계획
## Kubernetes & Application Integration

## 1. 목적과 문서 경계

이 문서는 「石나가는 판단」 1차 프로젝트의 Kubernetes & Application Integration 영역에서 **무엇을 시험하고, 무엇을 측정하며, 어떤 근거로 PASS/FAIL을 판정할지** 정의한다.

문서 간 책임은 다음과 같이 구분한다.

```text
09 = What / Why / Responsibility / Gate
11 = Pre-check / Apply / Verify / Re-run / Recovery / Rollback
12 = Test Case / Measurement / PASS·FAIL / Evidence
```

12는 11의 명령과 실행 절차를 다시 복사하지 않는다. 실제 실행 순서는 11을 참조하고, 본 문서는 시험 목적·조건·측정값·판정 기준·Evidence를 소유한다.

상위 기준:

- [`02_SeokPan_핵심_문제_및_검증_목표.md`](../02_SeokPan_핵심_문제_및_검증_목표.md)
- [`07_SeokPan_확장_호환형_MVP_도출.md`](../07_SeokPan_확장_호환형_MVP_도출.md)
- [`09_MVP_실행·통합_실시설계_Kubernetes_Application_Integration_정태훈.md`](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Kubernetes_Application_Integration_정태훈.md)
- [`10_GitHub_협업_및_Repository_운영.md`](../10_GitHub_협업_및_Repository_운영.md)
- [`11_MVP_구축·자동화_Runbook_Kubernetes_Application_Integration_정태훈.md`](../11_MVP_구축·자동화_Runbook/11_MVP_구축·자동화_Runbook_Kubernetes_Application_Integration_정태훈.md)
- [`PROJECT_CHANGES.md`](../PROJECT_CHANGES.md)
- [`MVP_IMPLEMENTATION_BASELINE.md`](../MVP_IMPLEMENTATION_BASELINE.md)
- `seokpan-app`, `seokpan-gitops`, `seokpan-infra`의 최신 `main`, Issue, PR, Runtime Evidence

현재 상태는 계속 변하므로 실제 Test 실행 직전에는 관련 Repository의 최신 Revision과 열린 Issue/PR을 다시 확인한다.

---

## 2. 상위 검증축과 역할 연결

02의 검증축을 대체하지 않는다. 본 문서의 Test Case는 아래 상위 검증축을 실제 실행 단위로 구체화한다.

| 상위 ID | 검증축 | 02의 핵심 판정 원칙 | 본 역할과의 관계 |
| --- | --- | --- | --- |
| M-01 | 동시성 정확성 | 정상 Turn Move 1개, Pass Turn Move 0개, 중복 Result/Rating 0건, stale 요청 오염 0건 | 핵심 책임 |
| M-02 | Vote 처리 성능 | 단계 부하로 처리량·p95/p99·오류율 변화를 측정하고 병목 Baseline 확인 | 핵심 책임 |
| M-03 | Backend 장애 복구 | 상태 복원, 잘못된 사용자 패배 판정 0건, 복구시간 측정 | 핵심 책임 |
| M-04 | DR | Backup/Restore·데이터 무결성·RTO·RPO | Data/Storage Recovery 담당과 Cross-role |
| M-05 | Ansible 개선 | 수동 대비 구축·재구축 시간·직접 개입·실패 Task 비교 | Network/Infra Automation 담당과 Cross-role |

본 역할은 M-04/M-05의 주 측정 Owner가 아니다. 다만 Application Consumer가 정상 복원되었는지, 복구 후 서비스가 다시 통합되는지 확인하는 Cross-role Evidence를 제공한다.

### 2.1 추가 검증축

02의 M-01~M-05 외에도 07/09/11에서 본 역할이 직접 소비하는 다음 실행 검증을 별도로 추적한다.

```text
Application Artifact Traceability
GitOps Sync / Self-Heal / Rollback
Frontend Runtime
HTTPRoute / HTTPS / WSS
Browser E2E
Application Metrics / Logs
Config / Secret / CA Consumer 반영
```

이 항목들은 M-01~M-05를 대체하지 않으며 MVP Acceptance를 구성하는 통합 Evidence다.

---

## 3. 측정 원칙

### 3.1 결과와 계획을 분리한다

상태는 다음 값으로 구분한다.

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

다음을 같은 뜻으로 사용하지 않는다.

```text
Manifest 존재 ≠ Runtime Running
Runtime Running ≠ Integration Validated
Image Pushed ≠ GitOps Updated ≠ Argo Synced ≠ Pod Ready
Platform Running ≠ Application Consumer Validated
```

### 3.2 근거 없는 목표값을 만들지 않는다

다음 값은 구현·실험 근거 없이 임의 확정하지 않는다.

- 동시 사용자 수
- Vote events/s 목표값
- CPU/Memory HPA Target
- `minReplicas` / `maxReplicas`
- p95/p99 목표 지연
- 허용 오류율
- RTO/RPO 목표값
- 개선률

성능은 목표 물리환경에서 부하를 단계적으로 증가시키고 **지연·오류가 급격히 악화되는 지점과 자원 병목**을 Baseline으로 정의한다.

### 3.3 핵심 KPI와 진단 지표를 구분한다

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

CPU 80%와 같은 단일 자원 값 자체를 프로젝트 성공 기준으로 사용하지 않는다.

### 3.4 현재 MVP 범위를 우선한다

02에는 AI 판세 분석이 존재하는 조건의 영향을 함께 확인한다는 원칙이 있지만, 현재 1차 MVP에서는 별도 ANALYSIS Runtime을 배포하지 않는다.

따라서 현재 12의 M-02 핵심 PASS/FAIL은 **Vote/Game 핵심 경로**를 기준으로 한다. 향후 ANALYSIS가 실제 Runtime에 포함되면 동일 부하 조건에서 분석 비활성/활성 Before→After를 별도 Test Run으로 추가한다.

---

## 4. Evidence 규격

각 Test Run은 최소 다음 Metadata를 남긴다.

```text
Test Case ID
Run ID
실행 시작/종료 시각
Operator
Git Repository / Commit SHA
GitOps Revision
Application Commit SHA
Backend / Frontend Image Digest
Kubernetes Context / Namespace
관련 Issue / PR
선행조건 상태
실행 결과
PASS / FAIL / BLOCKED / NOT TESTED
실패 원인 또는 중단 이유
수동 개입 단계
후속 조치
```

가능하면 다음 Evidence를 함께 저장한다.

```text
실행 명령 또는 Runbook 참조
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
RTO / RPO — 해당 Test에서 측정하는 경우
```

### 4.1 Evidence 금지 항목

다음은 Evidence에 기록하지 않는다.

- Password
- 전체 DB URL
- Token
- Private Key
- kubeconfig Credential
- Kubernetes Secret 실제 값

Secret은 Object 존재·Key 이름·Consumer 참조까지만 기록한다.

### 4.2 Run ID

권장 형식:

```text
KAI-<TEST-ID>-YYYYMMDD-HHMM-<short-sha>
```

예:

```text
KAI-RUN-01-20260914-2030-a1b2c3d
```

Run ID는 Evidence 디렉터리, Issue Comment, 측정 결과 파일에서 동일하게 사용한다.

### 4.3 실패 Run 보존

FAIL/BLOCKED Run을 수정하여 PASS 결과로 덮어쓰지 않는다.

```text
실패 Run 보존
→ 원인/조치 기록
→ 새 Run ID 발급
→ 동일 Test Case 재실행
```

---

## 5. Dependency Evidence 재사용 기준

이미 검증된 Platform Capability는 관련 Revision이 바뀌지 않았다면 매 Test Run마다 불필요하게 재구축하지 않는다.

재사용 가능 예:

- Kubernetes Cluster / Calico / CoreDNS
- Namespace / RBAC
- Argo CD Root/Child Application 구조
- Redis StatefulSet / PVC Persistence
- Backend DB Runtime/Migration Secret 공급 자동화
- MaxScale TLS Listener / CA 계약
- One-shot Migration 실행 자산의 정적/API 검증
- Gateway Platform HTTPS/TLS

다만 아래 경우에는 재검증한다.

```text
관련 Manifest / Playbook / Image 변경
Version 변경
Secret / CA 변경
Cluster 재구축
장애/복구 후
실제 Consumer 연결 최초 수행
이전 Evidence 대상 Revision과 현재 Revision 불일치
```

Dependency Evidence 재사용은 “현재 Application Integration까지 검증됨”을 의미하지 않는다.

---

## 6. 현재 검증 기준 상태

문서 작성 시점의 Current State:

| 영역 | 상태 | 12 처리 |
| --- | --- | --- |
| Kubernetes / Calico / CoreDNS | Validated | Dependency Evidence 재사용 가능 |
| Namespace / RBAC / Argo CD | Validated | Dependency Evidence 재사용 가능 |
| Redis Runtime / Persistence | Validated | Backend Consumer 검증은 별도 |
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

상태가 바뀌면 실제 Repository / Issue / PR / Runtime Evidence를 우선하여 본 표를 현행화한다.

---

## 7. Test Case Traceability Matrix

| Test Case | 상위 검증축/목적 | 09 Gate | 11 연결 | 현재 상태 |
| --- | --- | --- | --- | --- |
| KAI-PRE-01 Dependency Snapshot | 공통 | A~C | 4~6 | Defined |
| KAI-DEL-01 Artifact→GitOps→Argo Traceability | CI/CD Integration | B~D | 5 / 8 / 13 | Blocked |
| KAI-MIG-01 Migration Gate | Runtime/Data Integrity 선행 | C→D | 7 | Not Tested |
| KAI-RUN-01 Backend 1 Replica Provider Runtime | M-03 선행 / Runtime | D | 8 | Blocked |
| KAI-RUN-02 Backend 2 Replica Shared Runtime | M-01 / M-03 | E | 9 | Blocked |
| KAI-CON-01 동시성 정확성 | M-01 | E~G | 9 | Blocked |
| KAI-PERF-01 Vote 단계 부하 | M-02 | E~G | 9~10 | Blocked |
| KAI-REC-01 Backend 장애 복구 | M-03 | E~G | 9 / 15 / 17 | Blocked |
| KAI-HPA-01 HPA / Workload Distribution | M-02 | E | 10 | Not Implemented |
| KAI-FE-01 Frontend Runtime | 통합 | E | 11 | Blocked |
| KAI-RT-01 HTTPRoute / HTTPS / WSS | 통합 / M-03 | F | 12 | Not Implemented |
| KAI-E2E-01 Browser First Success | M-01 / M-03 | F~G | 11~12 | Blocked |
| KAI-CD-01 Argo Self-Heal | GitOps | G | 13 | Planned |
| KAI-CD-02 Git Revert Rollback | M-03 / GitOps | G | 13 / 17 | Planned |
| KAI-OBS-01 Application Metrics | C-06 | G | 14 | Blocked |
| KAI-OBS-02 Application Logs | C-06 | G | 14 | Blocked |
| KAI-CFG-01 Config / Secret / CA Consumer 반영 | 공통 | D~G | 16~17 | Planned |
| KAI-DR-01 Restore 후 Application 정상화 | M-04 Cross-role | G | 17 | Planned |

---

# 8. Test Case 상세

## KAI-PRE-01 — Dependency Snapshot

### 목적

Application Runtime 시험 전에 선행 Platform과 작업 대상 Revision을 고정한다.

### 측정/기록

- Node Ready 상태
- Argo Application Target Revision / Sync / Health
- Backend/Frontend Desired Replica/Image 상태
- Redis Runtime 상태
- Secret Object 및 Key 이름
- CA ConfigMap 존재
- Backend/Frontend Image Digest
- App/GitOps/Infra Commit SHA

### PASS

- 시험에 필요한 Dependency가 모두 존재한다.
- 시험 대상 Revision과 Evidence Metadata가 일치한다.
- Blocker가 있으면 다음 Test Case로 진행하지 않는다.

### FAIL/BLOCKED

- Required Dependency 부재
- Revision 불일치
- Secret/CA/Runtime Endpoint 부재
- A-10 완료 전 Backend Runtime 강제 시험 시도

### Evidence

- Pre-check snapshot
- Commit/Digest list
- Argo Application snapshot

---

## KAI-DEL-01 — Application Artifact → GitOps → Argo Traceability

### 목적

Application Source Commit에서 생성된 Image가 Harbor Digest로 식별되고, GitOps Desired State 변경을 거쳐 Argo CD와 실제 Pod까지 같은 Artifact로 연결되는지 검증한다.

Jenkins/Harbor의 구축 Owner는 Delivery 영역이지만, 정태훈 역할은 Application Consumer와 GitOps Desired State의 연결을 검증한다.

### Traceability Chain

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

### 측정/기록

- App Commit SHA
- Jenkins Run ID/시작/종료
- Build/Push 소요시간 — 제공 가능한 경우
- Harbor Tag/Digest
- GitOps PR 번호/Commit
- PR Merge 시각
- Argo Sync/Healthy 시각
- Pod Ready 시각
- Pod ImageID

### PASS

- App Commit과 Jenkins Run이 연결된다.
- Harbor Digest가 확인된다.
- GitOps Desired State가 해당 Digest를 고정한다.
- Argo CD가 해당 GitOps Revision을 Sync한다.
- 실제 Pod ImageID가 승인된 Digest와 일치한다.
- `latest` 또는 `git-pending`을 실제 Runtime Artifact로 사용하지 않는다.

### FAIL

- Commit→Digest 연결 불가
- GitOps가 다른 Digest를 사용
- Argo Sync 후 Pod가 다른 Image 실행
- Artifact Evidence 누락으로 실행 Image를 식별할 수 없음

### Evidence

- Jenkins Run link
- Harbor digest evidence
- GitOps PR/Commit
- Argo Sync/Health
- Pod ImageID
- Build/Push/Sync/Ready Timeline

---

## KAI-MIG-01 — 승인형 Migration Gate

### 목적

Backend Runtime을 시작하기 전에 실제 DB Revision과 승인된 Migration Action이 일치하고, 수행 후 데이터/복제 상태가 정상인지 검증한다.

이 Test Case는 **Schema Migration Runtime Gate**이며 M-04 DR 자체를 대신하지 않는다.

### 선행조건

- A-10을 포함한 승인된 Backend Image Digest
- DB Backup/Restore 준비 상태
- Replication 확인
- Migration Secret 준비
- CA 준비
- Active Migration Job 없음
- 실행 승인 Reference

### 측정/기록

- 실행 전 Alembic Revision
- 결정된 Action: `current` / `stamp-baseline` / `upgrade-head`
- 실행 Job 이름
- Job 시작/종료 시각
- Job terminal status
- 실행 후 Alembic Revision
- Primary/Replica Replication 상태
- MaxScale Read/Write 확인
- 표본 데이터 보존 확인

### PASS

- DB Audit 결과와 선택 Action이 일치한다.
- 승인된 Job이 1회 실행되고 성공 종료한다.
- 기대 Revision에 도달한다.
- Replication이 정상이다.
- 기존 표본 데이터가 보존된다.
- Runtime DB User와 Migration DB User 권한 경계가 유지된다.

### FAIL

- Active Job 중복
- 승인되지 않은 Mutation
- Job Failed
- 기대 Revision 불일치
- Replication 오류
- 데이터 손실/변형
- 일반 Backend가 Migration Credential을 소비함

### Evidence

- Approval Reference
- Rendered Job checksum
- Job status/log
- Revision Before/After
- Replication snapshot
- 표본 데이터 checksum 또는 비교 결과

실제 DB Restore/RTO/RPO의 주 Owner는 M-04 담당 영역이며, 본 Test Case는 Runtime 진입 Gate에 필요한 DB 정상성까지만 확인한다.

---

## KAI-RUN-01 — Backend 1 Replica Provider First Runtime

### 목적

Production Backend가 Memory Provider 없이 실제 MariaDB·Redis Provider를 사용하여 1 Replica로 정상 기동하고, Provider-aware readiness를 통과하는지 검증한다.

### 선행조건

- A-10 Production Provider Integration 완료
- KAI-DEL-01의 Backend Artifact Traceability 성립
- KAI-MIG-01 PASS
- GitOps #90 Origin JSON 계약 완료
- Runtime Secret / CA / Redis 준비
- Backend Digest Pinning
- `replicas: 1` Desired State

### 측정/기록

- Argo Sync/Health
- Deployment desired/ready replicas
- Pod Scheduling / Image Digest
- Startup/Liveness/Readiness
- MaxScale TLS 연결
- Identity DB 연결
- Game DB 연결
- Redis Service DNS 연결
- Provider readiness 결과
- 최소 기능 First Success 결과

### PASS

- Backend Pod 1개가 Ready다.
- 실행 Image Digest가 승인된 Digest와 일치한다.
- Startup/Live/Ready 의미가 구분되어 정상 상태다.
- Identity/Game DB 연결이 TLS 계약으로 성공한다.
- Redis Consumer 연결이 성공한다.
- 일반 Backend는 Migration Credential을 소비하지 않는다.
- production 성공 경로에 Memory Provider가 남지 않는다.

### FAIL

- ImagePull/Config/Secret 오류
- Provider 연결 실패
- TLS 우회 또는 CA 미적용
- Readiness가 Provider 장애에도 Ready를 유지함
- Migration Credential 소비
- Runtime 기동을 위해 Memory Provider로 fallback

### Evidence

- Deployment/Pod snapshot
- ImageID/Digest
- Health 결과
- DB/Redis non-secret connection result
- Application log excerpt
- Argo Sync/Health

---

## KAI-RUN-02 — Backend 2 Replica Shared Runtime

### 목적

1 Replica에서 검증된 Backend를 2 Replica로 확장했을 때 권위 상태가 Pod-local Memory에 분리되지 않고 Redis/DB 공유 상태를 통해 수렴하는지 확인한다.

### 선행조건

- KAI-RUN-01 PASS
- A-10 Shared State / Realtime / Presence / Runner 구현 완료
- 두 Replica가 동일 Image Digest 사용

### 측정/기록

- Pod 2개 Ready
- Pod별 Instance ID
- Worker 배치 상태
- 동일 Room/Game 요청의 Replica 간 결과
- Redis shared state
- Pub/Sub 전달
- Reconnect 결과
- Runner Lease / 중복 실행 상태

### PASS

- 두 Replica가 동일 권위 상태로 수렴한다.
- Process-local Memory가 권위 상태가 아니다.
- Pub/Sub 누락/재연결 시 Snapshot 재조회로 수렴한다.
- Runner의 동일 작업 중복 마감이 발생하지 않는다.
- 실제 WebSocket 객체 Registry는 Pod-local이어도 사용자 상태는 공유 계약과 일치한다.

### FAIL

- Replica별 Room/Game 상태 불일치
- 동일 요청이 Replica에 따라 다른 권위 결과 생성
- Runner 중복 마감
- Reconnect 후 stale 상태 유지
- Pub/Sub을 영구 권위 저장소로 가정하여 복구 불가

### Evidence

- 두 Pod 로그/Instance ID
- Redis state snapshot
- 동일 요청 결과 비교
- Runner execution timeline

---

## KAI-CON-01 — M-01 동시성 정확성

### 목적

짧은 시간의 Vote 등록·변경·삭제·마감·재시도와 stale 요청이 겹쳐도 Move/Result/Rating이 중복·유실되지 않는지 검증한다.

### 핵심 측정값

```text
정상 Turn당 확정 Move 수
Pass Turn당 확정 Move 수
중복 GameResult 반영 건수
중복 Rating 반영 건수
stale 요청으로 인한 상태 변경 건수
동일 idempotency request의 중복 반영 건수
```

### PASS — 상위 M-01 기준

```text
정상 Turn 확정 Move = 1개
Pass Turn 확정 Move = 0개
중복 GameResult = 0건
중복 Rating = 0건
stale 요청의 잘못된 상태 변경 = 0건
```

추가로 동일 idempotency request가 여러 Replica에 도달해도 권위 결과는 한 번만 반영되어야 한다.

### FAIL

위 0건 조건 중 하나라도 위반하거나 정상 Turn의 Move 수가 1이 아닌 경우 FAIL.

### 부하 규모

동시 요청 수를 임의 고정하지 않는다. 실제 환경에서 재현 가능한 단계 부하를 사용하고 각 단계의 동시성 오류 여부를 함께 기록한다.

### Evidence

- 요청 ID / event_id / state_version
- Turn/Move snapshot
- GameResult/RatingHistory 비교
- DB/Redis 상태 비교
- Application logs

---

## KAI-PERF-01 — M-02 Vote 처리 단계 부하

### 목적

투표 집중 구간에서 처리량·지연·오류율의 변화와 병목 지점을 측정한다.

### 측정값

```text
Vote events/s
p50 / p95 / p99 latency
Error rate
Timeout count
CPU / Memory
Pod Replica 수
Redis latency/connection 상태
DB connection/query 진단값
WebSocket connection 수
Active Room/Game
```

### 부하 방식

고정된 “목표 동시 사용자 수”를 먼저 정하지 않는다.

```text
낮은 부하 Baseline
→ 단계 증가
→ 각 단계 안정 구간 측정
→ p95/p99·오류율·자원 변화 관찰
→ 급격한 성능 악화 지점 확인
→ 병목 원인 추적
```

### PASS/판정

이 Test Case의 1차 목적은 **실제 물리환경의 Baseline을 측정하는 것**이다.

PASS 조건:

- 단계별 처리량·p95/p99·오류율이 누락 없이 측정된다.
- M-01 정확성 조건을 깨지 않는다.
- 병목이 발생한 단계와 진단 지표를 식별할 수 있다.

현재 1차 MVP에서 별도 ANALYSIS Runtime은 배포하지 않으므로 AI 판세 분석 조건을 현재 PASS의 필수 전제로 두지 않는다. 향후 ANALYSIS가 실제 Runtime에 포함되면 동일 부하 Before→After를 별도 추가한다.

### Before/After

구조 변경이 실제로 발생한 경우에만 동일 조건으로 비교한다.

```text
Static Replica Baseline
→ Replica/HPA 구조 변경
→ 동일 부하 재실행
→ 처리량/p95/p99/Error 비교
```

### Evidence

- Load generator config
- Raw result
- Latency percentile
- Error breakdown
- Prometheus export
- Grafana snapshot
- Pod/Node resource snapshot

---

## KAI-REC-01 — M-03 Backend 장애 복구

### 목적

단일 Backend Replica 장애가 사용자 패배로 오판되지 않고, 재연결 또는 다른 Replica를 통해 Room/Game/Turn/Board 상태가 복원되는지 검증한다.

### 선행조건

- KAI-RUN-02 PASS
- Reconnect/Snapshot 복구 계약 구현
- 장애 주입 범위와 중단 조건 사전 합의

### 측정값

```text
장애 주입 시각
첫 실패 요청 시각
재연결 성공 시각
서비스 복구 시각
상태 복구 완료 시각
실패 요청 수
재연결 성공/실패
잘못된 사용자 패배 판정 건수
Room/Game/Turn/Board Before/After
```

### PASS — 상위 M-03 기준

- 단일 Backend 장애가 사용자 몰수패/공동 패배로 처리되지 않는다.
- 잘못된 사용자 패배 판정 = 0건.
- 재연결 후 권위 상태가 복원된다.
- 복구시간을 측정할 수 있다.
- 장애 전후 Room/Game/Turn/Board가 일관된 상태로 수렴한다.

RTO 목표값은 근거 없이 사전 확정하지 않고 실제 측정값을 Baseline으로 남긴다.

### Evidence

- 장애 Timeline
- Pod event
- Client reconnect log
- State snapshot Before/After
- Application/Redis logs
- Recovery duration calculation

---

## KAI-HPA-01 — HPA / Workload Distribution

### 상태

현재 HPA/Resource Metrics 자산은 구현 완료 상태가 아니므로 **Planned / Not Tested**다.

### 선행조건

- KAI-RUN-01 PASS
- KAI-RUN-02 PASS
- Resource Request/Limit 정의
- Resource Metrics 준비
- 11의 HPA ↔ Argo CD Replica Ownership 계약 확정
- HPA Manifest Merge

### 측정값

```text
부하 단계
HPA target/current metric
Desired / Current Replica
Scale-out 시작/완료 시각
Scale-in 시작/완료 시각
p95/p99
Error rate
Pod Ready 상태
Argo Diff/Sync 상태
```

### PASS

- HPA가 구성된 Metrics를 실제로 읽는다.
- 설정된 범위 안에서 Replica를 변경한다.
- Scale-out/in 중 서비스 가용성이 유지된다.
- Argo CD와 HPA가 `spec.replicas`를 두고 반복 경쟁하지 않는다.
- Replica 변경 동안 M-01 정확성을 깨지 않는다.

CPU Target·Replica 범위 자체의 적절성은 실제 단계 부하 결과를 근거로 판단한다.

### FAIL

- Metrics Unknown
- HPA/Argo 반복 Diff 또는 Replica 되돌림
- Scale 중 지속 오류 증가/가용성 상실
- M-01 오류 발생

---

## KAI-FE-01 — Frontend Runtime

### 목적

승인된 Frontend Image가 GitOps를 통해 실제 Runtime으로 활성화되고 Service가 정상 동작하는지 확인한다.

### 선행조건

- 09의 Critical Path 기준 Backend Scale-out / HPA·Workload Distribution 단계 판정 완료
- Frontend Image Digest 확보
- Container Runtime Smoke 확인
- `apps-frontend` Child Application 정상

HPA가 일정상 Deferred되는 경우에는 이를 PASS로 가장하지 않고 별도 Go/No-Go 결정과 근거를 남긴 뒤 Frontend 단계로 진행한다.

### 측정/기록

- GitOps Revision
- Frontend Image Digest
- Deployment desired/ready
- Pod state
- Service Endpoint
- SPA fallback
- Container health

### PASS

- 승인된 Digest가 실행된다.
- Frontend Pod/Service가 정상이다.
- SPA 기본 경로가 정상 응답한다.
- Runtime 성공을 Browser E2E 성공과 혼동하지 않는다.

---

## KAI-RT-01 — Application HTTPRoute / HTTPS / WSS

### 상태

현재 Application HTTPRoute는 **Not Implemented / Not Tested**다.

### 선행조건

- KAI-RUN-01 PASS
- KAI-FE-01 PASS
- Gateway Platform 정상
- Application HTTPRoute Manifest Merge

### 검증 경로

```text
/       → frontend:8080
/api/v1 → backend:8000
/ws/v1  → backend:8000
```

### 측정/기록

- Gateway/HTTPRoute Conditions
- `Accepted`
- `ResolvedRefs`
- HTTPS response
- API response
- WSS handshake
- WebSocket message exchange
- Forwarded Header
- Cookie/CORS
- DELETE Request Body
- 외부 비공개 Endpoint 확인

### PASS

- Route Condition이 정상이다.
- HTTPS Frontend 접근 성공.
- `/api/v1`이 Backend로 전달된다.
- `/ws/v1` WSS Upgrade 및 메시지 교환 성공.
- CORS/Cookie/Forwarded Header 계약이 실제 Browser 조건에서 정상.
- 내부 Health/Metric Endpoint가 의도치 않게 외부 공개되지 않는다.

---

## KAI-E2E-01 — Browser First Success

### 목적

팀원 PC의 Browser에서 실제 공개 Hostname을 사용하여 최소 사용자 흐름이 Frontend→Gateway→Backend→DB/Redis까지 연결되는지 검증한다.

### 최소 사용자 흐름

실제 구현 범위에 맞춰 다음 중 구현 완료된 MVP 핵심 경로를 사용한다.

```text
Frontend 접속
→ 회원/세션
→ Lobby/Room
→ 팀 선택/Ready
→ Game/Vote
→ Move 확정
→ Realtime Update
→ 결과 확인
```

미구현 기능을 Test 성공 조건에 억지로 포함하지 않는다.

### PASS

- 실제 Browser에서 HTTPS/WSS 사용.
- 같은 Origin 계약 유지.
- 핵심 상태가 재접속 후에도 일관됨.
- Backend/Redis/DB 실제 Provider 경로 사용.
- Fake/Memory Provider E2E와 구분됨.

### Evidence

- Browser timestamp
- Network trace 요약
- Backend Instance ID
- Application logs
- Result/Move state
- Screenshot

---

## KAI-CD-01 — Argo CD Self-Heal

### 목적

Git Desired State가 Source of Truth이며, 명시적인 검증용 Live Drift가 발생했을 때 Self-Heal로 정상 상태에 수렴하는지 확인한다.

### 안전 원칙

- Production 데이터 변경을 사용하지 않는다.
- 쉽게 복구 가능한 비영속/무해한 필드를 Test 대상으로 사용한다.
- 시험 전 정상 Git Revision을 고정한다.

### 측정값

```text
Drift 주입 시각
OutOfSync 감지 시각
Self-Heal 시작/완료 시각
최종 Sync/Health
Live Object Before/After
```

### PASS

- Git을 변경하지 않은 Live Drift가 감지된다.
- Self-Heal 후 Live State가 Git Desired State와 다시 일치한다.
- 서비스 가용성에 불필요한 영향을 만들지 않는다.

---

## KAI-CD-02 — Git Revert Rollback

### 목적

문제가 있는 GitOps Revision을 이전 Known-Good Revision으로 되돌리고 Application Runtime이 복구되는 시간을 측정한다.

### 원칙

DB Schema Migration은 단순 Git Revert 대상이 아니다. 이 Test Case는 Application/GitOps Desired State Rollback을 검증한다.

### 측정값

```text
문제 Revision 인지 시각
Revert PR 생성/승인/Merge 시각
Argo Sync 시각
Pod Ready 시각
서비스 정상화 시각
총 Recovery 시간
수동 개입 단계
```

### PASS

- Git `main`이 Known-Good Desired State로 복귀한다.
- Argo CD가 해당 상태를 Sync한다.
- Runtime이 Known-Good Image/Config/Replica 상태로 복구된다.
- 임의 Cluster patch를 장기 상태로 남기지 않는다.

---

## KAI-OBS-01 — Application Metrics

### 상태

Application ServiceMonitor는 현재 `.pending`이며 GitOps #91에서 Runtime 활성화 계약을 추적한다.

### 선행조건

- Backend Runtime Running
- `/metrics` 실제 제공
- Backend Service `metadata.labels` 계약 확정
- ServiceMonitor selector 정합
- `.pending` 제거 및 GitOps Merge

### 측정값

```text
ServiceMonitor 존재
Service metadata.labels / ServiceMonitor selector
Prometheus Target State
Target scrape error
Application metric sample
Pod/instance label
```

### PASS

- Prometheus Target = UP.
- ServiceMonitor가 의도한 Backend Service의 `metadata.labels`를 선택한다.
- 실제 Application Metric이 조회된다.
- Platform Prometheus Running을 Application Metrics 성공으로 대신하지 않는다.

### FAIL

- Target 미생성
- selector/metadata.labels 불일치
- Target Down
- `/metrics` 미제공

---

## KAI-OBS-02 — Application Logs

### 목적

Backend/Frontend Runtime의 stdout/stderr 로그가 Alloy를 통해 Loki에 수집되고 Pod Identity를 추적할 수 있는지 확인한다.

### 현재 Platform 계약

Alloy는 Kubernetes Pod를 discovery하고 다음 라벨을 사용한다.

```text
namespace
pod
container
node_name
```

### 측정/기록

- Application Pod 이름
- Container 이름
- Node
- 해당 시각의 Application log event
- Loki 조회 결과
- label 일치 여부
- 수집 지연

### PASS

- `application` Namespace의 실제 Runtime Pod 로그가 Loki에서 조회된다.
- `namespace/pod/container/node_name`이 실제 Pod와 일치한다.
- 민감정보가 Raw Log에 노출되지 않는다.
- Alloy/Loki Platform Running과 실제 Application Log 수집 성공을 분리해서 증명한다.

### 민감정보 검사

실제 Secret 값을 문서/명령에 노출하여 검색하지 않는다. 대신 다음과 같은 구조적 패턴을 확인한다.

```text
Password 값 직접 출력
전체 DB URL Credential 포함 출력
Token / Authorization header 원문 출력
Private Key 출력
```

---

## KAI-CFG-01 — Config / Secret / CA Consumer 반영

### 목적

GitOps/Infra에서 ConfigMap·Secret·CA가 변경된 경우 실행 중 Pod가 자동으로 변경값을 읽는다고 잘못 가정하지 않고, 새 Pod에 의도한 값이 반영되었는지 확인한다.

### 핵심 경계

- Secret env는 기존 프로세스 환경변수에 자동 재주입되지 않는다.
- ConfigMap `envFrom`도 기존 프로세스에 자동 반영되지 않는다.
- `subPath`로 Mount한 CA 파일은 ConfigMap 변경만으로 기존 Container 파일이 갱신되지 않는다.

### 측정/기록

- 변경 전 Pod UID
- 변경 Git/Infra Revision
- Rollout 발생 여부
- 변경 후 Pod UID
- 새 Pod의 Config/CA fingerprint 또는 non-secret setting
- Readiness

### PASS

- 필요한 경우 새 Pod가 생성된다.
- 새 Pod가 현재 Desired State를 소비한다.
- CA 변경 시 새 Pod가 기대 CA fingerprint를 사용한다.
- 직접 `kubectl rollout restart`만 수행하고 Git 기록 없이 끝내지 않는다.

---

## KAI-DR-01 — Restore 후 Application 정상화 Cross-role

### 목적

Data/Storage Recovery 담당이 수행한 Backup/Restore 이후 Application Consumer가 정상 데이터와 상태를 다시 사용할 수 있는지 통합 검증한다.

### 주 Owner

M-04 DR의 Backup/Restore, RTO/RPO 측정은 Data/Storage Recovery 담당이 주 Owner다.

### 본 역할 확인

- Backend가 복원된 DB Endpoint에 정상 연결
- Redis/DB 상태 경계 정상
- Member/MemberStats/GameResult/Move/RatingHistory 표본 조회
- Application Readiness 정상
- Browser/API 핵심 흐름 정상

### PASS

- Restore Evidence와 Application Consumer Evidence가 동일 Run/Revision으로 연결된다.
- 표본 데이터 무결성 검증이 PASS다.
- Application이 복원 데이터로 정상화된다.

본 Test Case는 다른 역할의 RTO/RPO 값을 임의로 재정의하지 않는다.

---

# 9. Before → Change → After 측정 설계

프로젝트 주장을 수치로 설명할 필요가 있는 항목은 가능한 경우 동일 조건 Before/After를 사용한다.

## 9.1 Replica / HPA

```text
Before: 검증된 Static Replica 조건
Change: Replica Scale-out 또는 HPA
After: 동일 부하 재실행
```

비교:

```text
Vote events/s
p95 / p99
Error rate
Resource usage
Replica 수
M-01 오류 여부
```

## 9.2 장애 복구

```text
Before: 정상 상태 Snapshot
Change: 통제된 Backend 장애
After: 재연결/복구 후 Snapshot
```

비교:

```text
상태 일관성
복구시간
실패 요청
잘못된 패배 판정
```

## 9.3 GitOps Rollback

```text
Before: Known-Good Revision
Change: 검증 가능한 문제 Revision / Drift
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

## 9.4 Artifact Delivery

```text
Before: App Commit 확정
Change: Build / Push / GitOps Update
After: Argo Sync / Pod Ready
```

비교/기록:

```text
Build 시간
Push 시간
GitOps PR/Merge 시간
Argo Sync 시간
Pod Ready 시간
Commit → Digest → Pod Traceability
```

---

# 10. Test 실행 순서

09의 Critical Path를 우선한다.

```text
KAI-PRE-01
→ KAI-DEL-01 Backend Artifact
→ KAI-MIG-01
→ KAI-RUN-01 Backend 1 Replica
→ KAI-RUN-02 Backend 2 Replica
→ KAI-CON-01
→ KAI-PERF-01
→ KAI-REC-01
→ KAI-HPA-01 HPA / Workload Distribution
→ KAI-DEL-01 Frontend Artifact
→ KAI-FE-01 Frontend Runtime
→ KAI-RT-01 HTTPRoute / HTTPS / WSS
→ KAI-E2E-01 Browser First Success
→ KAI-CD-01 / KAI-CD-02
→ KAI-OBS-01 / KAI-OBS-02
→ KAI-CFG-01
→ Cross-role KAI-DR-01
```

실제 기능 구현 의존성에 따라 일부 비파괴 Test는 병행할 수 있다. 다만 다음 Gate는 깨지 않는다.

```text
A-10 미완료 → Backend Runtime 실행 금지
Backend 1 Replica 미검증 → 2 Replica/HPA 진행 금지
Backend 2 Replica Shared Runtime 미검증 → M-01/M-03 최종 판정 금지
HPA가 미구현/Deferred인데 이를 PASS로 표시 금지
Frontend/Backend Service 미준비 → Application HTTPRoute E2E 진행 금지
Metrics Endpoint/Service metadata label 미정 → ServiceMonitor 활성화 금지
```

HPA가 일정/Go-No-Go 결정으로 Deferred되는 경우에는 해당 상태와 근거를 명시하고 Frontend 단계 진행 여부를 별도 결정한다. Deferred를 PASS로 기록하지 않는다.

---

# 11. 실패/중단 판정

다음은 즉시 중단 조건이다.

- 데이터 손실 위험
- Migration 예상 외 Revision
- Replication 오류
- Secret/Token/Password 노출
- 잘못된 GameResult/Rating 반영
- M-01 정확성 오류
- Backend 장애가 사용자 패배로 잘못 확정됨
- HPA/Argo Replica ownership 경쟁으로 지속 변동
- Test가 의도한 Revision이 아닌 Image/Manifest를 사용함

중단 후:

```text
Run 상태 = FAILED 또는 BLOCKED
→ 원인 기록
→ 영향 범위 고정
→ 필요 시 11 Rollback/Recovery 수행
→ 새 Run ID로 재실행
```

실패 Run을 수정해서 PASS 기록으로 덮어쓰지 않는다.

---

# 12. 결과 기록 Template

각 Test Case 결과는 최소 다음 형식으로 남긴다.

```text
Test Case:
Run ID:
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

# 13. 최종 Acceptance 연결

09의 Gate G MVP Acceptance는 본 역할의 Test만으로 단독 완료되지 않는다.

최종 Acceptance에는 최소 다음이 함께 필요하다.

```text
Kubernetes/Application Integration Evidence
+ Data/Storage Recovery Evidence
+ Delivery/Observability Evidence
+ Network/External Infra Evidence
+ Browser/E2E Evidence
```

본 역할의 핵심 완료 조건:

- Application Artifact `Commit → Digest → GitOps → Argo → Pod` Traceability
- Migration Gate Evidence 존재
- Backend 1 Replica 실제 Provider First Runtime PASS
- Backend 2 Replica Shared Runtime PASS
- M-01 동시성 정확성 PASS
- M-02 단계 부하 Baseline 확보
- M-03 장애 복구 PASS 및 복구시간 측정
- Frontend Runtime PASS
- HTTPRoute/HTTPS/WSS PASS
- Browser E2E PASS
- GitOps Self-Heal/Rollback Evidence
- Application Metrics/Logs 실제 수집 Evidence
- HPA는 구현되면 PASS Evidence, 미구현/Deferred면 명확한 상태와 근거 유지

---

# 14. 문서 완료 기준

본 12 문서 자체의 완료 기준:

- 02의 M-01~M-05와 Traceability가 있다.
- 07의 Commit→Digest→Healthy 및 Metric/Log/Evidence 원칙과 충돌하지 않는다.
- 09 Critical Path와 Gate를 유지한다.
- 11 Runbook을 반복하지 않고 Test/Measurement/Evidence로 연결한다.
- 각 핵심 Test Case에 목적·측정값·PASS/FAIL·Evidence가 있다.
- 실제 실행하지 않은 항목은 결과처럼 작성하지 않는다.
- 성능 목표값·HPA 임계값·RTO/RPO를 근거 없이 만들지 않는다.
- 현재 MVP에서 제외된 ANALYSIS Runtime을 현재 PASS의 필수조건으로 만들지 않는다.
- Dependency Evidence 재사용과 Consumer 검증을 구분한다.
- Cross-role Owner 경계가 명확하다.
- Secret/Token/Private Key가 Evidence에 포함되지 않는다.
- 실패 Run을 보존하고 재실행은 새 Run ID로 관리한다.
- 실제 Repository/Runtime 상태와 재대조했을 때 추가 필수 보완사항이 없을 때 검토를 종료한다.
