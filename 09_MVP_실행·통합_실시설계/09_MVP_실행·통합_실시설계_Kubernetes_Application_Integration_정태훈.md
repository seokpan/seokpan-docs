# MVP 실행·통합 실시설계
## Kubernetes & Application Integration

## 1. 목적과 범위

이 문서는 「石나가는 판단」 1차 프로젝트에서 Kubernetes와 Application Integration 영역의 책임, Current State, Provider/Consumer 관계, 선행조건, Integration Gate와 완료 경계를 정의한다.

01~08은 Historical Baseline이며, 본 문서는 이를 다시 서술하지 않는다. Baseline 이후 확정 변경은 `PROJECT_CHANGES.md`, 공용 구현 기준은 `MVP_IMPLEMENTATION_BASELINE.md`, 실제 상태는 `seokpan-app`·`seokpan-gitops`·`seokpan-infra`의 현재 `main`, Issue, PR, Runtime Evidence를 우선한다.

- 상세 실행 명령·적용·복구·Rollback 절차: 11 구축·자동화 Runbook
- 공식 Test Case·측정·PASS/FAIL·Evidence 상세: 12 검증·측정 계획
- Repository·Directory·Issue·Branch·PR·Review Governance: [`10_GitHub_협업_및_Repository_운영.md`](../10_GitHub_협업_및_Repository_운영/10_GitHub_협업_및_Repository_운영.md)

## 2. 선행 기준과 상태 판정

직접 상위 기준은 다음 문서다.

- [`03_SeokPan_논리_역할_및_서비스_목록.md`](../01-08_기획·설계_Baseline/03_SeokPan_논리_역할_및_서비스_목록.md)
- [`04_SeokPan_기술_비교_및_논리_아키텍처.md`](../01-08_기획·설계_Baseline/04_SeokPan_기술_비교_및_논리_아키텍처.md)
- [`05_SeokPan_물리_아키텍처.md`](../01-08_기획·설계_Baseline/05_SeokPan_물리_아키텍처.md)
- [`06_SeokPan_Ansible_자동화_테스트_설계.md`](../01-08_기획·설계_Baseline/06_SeokPan_Ansible_자동화_테스트_설계.md)
- [`07_SeokPan_확장_호환형_MVP_도출.md`](../01-08_기획·설계_Baseline/07_SeokPan_확장_호환형_MVP_도출.md)
- [`08_SeokPan_1차_프로젝트_기획안.md`](../01-08_기획·설계_Baseline/08_SeokPan_1차_프로젝트_기획안.md)

특히 07의 역할·Provider/Consumer와 실행·협업 실시설계 인계를 직접 기준으로 사용한다.

상태 용어는 `Planned / Defined / Implemented / Merged / Running / Validated / Partial / In Progress / Blocked / Not Tested / Deferred`를 사용한다.

```text
Implemented ≠ Merged ≠ Running ≠ Validated
Platform Validated ≠ Provider Ready ≠ Consumer Wired ≠ Application Running ≠ Integration Validated
```

Gate 상태는 09의 실행·통합 진행 판정이며 12의 공식 시험 판정을 대신하지 않는다. 검증된 Pipeline이 존재해도 Source가 바뀌면 현재 Release Artifact는 다시 검증해야 한다.

## 3. 역할과 책임 경계

Kubernetes & Application Integration 영역은 Runtime Platform과 Application Runtime 사이의 연결을 담당한다.

주요 책임:

- Kubernetes Cluster·Namespace·RBAC·Argo CD 선행조건 확인
- Backend/Frontend Kubernetes Desired State
- Redis Runtime과 Application Consumer 연결
- Provider/Consumer Integration 조정
- Backend 1 Replica First Runtime
- 2 Replica Scale-out과 공유 Runtime State
- HPA와 Workload 분산 경로
- Frontend Runtime, HTTPRoute, HTTPS/WSS 연결
- Application Runtime의 Health·Metric·Log를 Observability가 실제 수집·조회할 수 있도록 연결하고 확인 기준을 정의

직접 소유하지 않는 영역:

| 영역 | 주요 Owner |
| --- | --- |
| Physical Network·VRouter·LB·공통 Ansible | Network / Infra Automation |
| MariaDB·MaxScale·NFS·Backup/Restore | Data / Storage / Recovery |
| Jenkins Runtime·Harbor·Observability Platform | Delivery / Observability |
| Application 기능 Source·Domain·Adapter | `seokpan-app` |
| GitHub Governance | 10 문서 |
| 실행 절차 | 11 문서 |
| 공식 시험·측정·Evidence | 12 문서 |

## 4. Kubernetes & Runtime Platform Current State

### 4.1 Cluster

- Control Plane 3대 / Worker 2대
- kubeadm / Calico
- Control Plane·Worker `Ready`
- kube-apiserver·CoreDNS·Calico 정상
- 기존 Runtime 기준 Kubernetes 회귀검증 완료

판정: `Validated`.

현재 동작 중인 Cluster 검증과 완전히 빈 환경에서의 전체 Clean Rebuild는 별도 상태로 구분한다.

### 4.2 Network / DNS

주요 Endpoint:

```text
harbor.seokpan.soldesk.store
db.seokpan.soldesk.store:3306
game.seokpan.soldesk.store
redis.platform.svc.cluster.local:6379
```

Application이 실제 소비하는 DB·Redis·Gateway 경로까지 Runtime에서 확인됐다.

### 4.3 Namespace / RBAC / Argo CD

주요 Namespace는 `application`, `platform`, `cicd`, `observability`, `storage-infra`다.

Namespace/RBAC는 Argo CD 관리 경로에 편입되어 있고 역할별 권한 경계가 검증됐다.

Argo CD는 Gateway, Redis, Namespace/RBAC, Jenkins, Observability, Storage, Backend, Frontend의 `Desired State`(Git에 선언된 Kubernetes 목표 상태)를 관리한다.

판정: `Validated`.

### 4.4 Gateway / External Access

Gateway Platform뿐 아니라 Application Route까지 실제 Runtime에서 연결됐다.

현재 확인된 범위:

- `/` → Frontend Service
- `/api/v1` → Backend Service
- `/ws/v1` → Backend WebSocket
- `game.seokpan.soldesk.store`
- Common VIP `10.1.93.90`
- HTTPS / WebSocket 경로
- Let's Encrypt Production 인증서
- Windows Host Browser와 프로젝트 Linux VM의 실제 접속

Evidence:

- `seokpan-infra#188` — Let's Encrypt DNS-01 발급, Gateway TLS Secret 적용, Linux VM·Windows Host HTTPS 접속 및 Guest Session/Lobby WebSocket 확인
- `seokpan-app#3` — A-10 Gateway/API/WebSocket Runtime Gate 완료 상태 연결

```text
Gateway Platform: Validated
Application HTTPRoute: Validated
HTTPS / API / WebSocket Runtime Gate: Validated
Full P4 Browser Acceptance: In Progress
```

외부 접속 Runtime Gate 통과를 전체 서비스 안정화 완료로 확대하지 않는다.

### 4.5 Redis

서비스용 Redis는 StatefulSet 1 Replica, Service 6379, NFS PVC, AOF(Append Only File, Redis 변경 내용을 순차 기록하는 persistence 방식), `appendfsync everysec` 기준으로 운영한다.

확인된 결과:

- Write/Read PASS
- Pod 재생성 뒤 데이터 유지 PASS
- Backend Production Runtime의 Redis Service DNS 연결 PASS
- Backend 1 Replica 이후 2 Replica에서 Session 공유 경로 PASS

Evidence:

- `seokpan-gitops#7` — Redis Runtime/Persistence 및 실제 Backend Consumer 연결 결과

```text
Redis Platform Runtime: Validated
Backend → Redis Consumer: Validated
2-Replica Session Sharing: Validated
```

Room/Vote/Game의 세부 동시성·복구 동작은 P4 Stabilization에서 별도로 검증한다.

### 4.6 Scale-out / HPA / Workload Distribution

Backend는 1 Replica Provider Gate를 통과한 뒤 2 Replica로 확장됐고, worker-01 / worker-02 분산 배치를 실제 Runtime에서 확인했다.

GitOps PR #94/#95에서 Backend 2 Replica, Worker 분산, PDB(PodDisruptionBudget, 계획된 중단 시 최소 가용 Pod 수를 보호하는 정책), RollingUpdate 구성이 반영됐다.

Frontend도 2 Replica Runtime으로 활성화됐다.

```text
Manual Runtime Scale-out: Validated
Backend 2 Replica / Worker Distribution: Validated
Frontend 2 Replica Runtime: Validated
HPA Integration: Not Implemented / Not Tested
```

HPA(Horizontal Pod Autoscaler, Metric에 따라 Pod Replica 수를 자동 조절하는 Kubernetes 기능)는 A-10 완료 조건으로 강제하지 않는다.

HPA를 도입할 경우 `Deployment.spec.replicas`를 Argo CD와 HPA 중 누가 최종적으로 변경·유지할지 먼저 확정하고 실제 Sync 동작을 검증해야 한다.

## 5. Application Integration Current State

### 5.1 Application Roadmap

```text
A-01~A-08: 기능·Headless·Frontend First Success — Completed
A-09: Container / Jenkins / Image Acceptance — Completed
A-10: Production Provider / GitOps / Kubernetes Runtime Integration — Completed
P4: Stabilization / Acceptance — In Progress
```

A-10 완료는 실제 Provider와 Kubernetes Runtime 연결이 성립했다는 의미다.

현재 진행 중인 `seokpan-app#76`의 서비스 안정화 작업까지 완료됐다는 뜻은 아니다.

### 5.2 Build / Artifact

Jenkins Image Pipeline은 현재 Source를 기준으로 Test → Build → Harbor Push → Scan → Health Smoke → Final Tag/Digest → SBOM/Provenance Evidence까지 생성한다.

현재 Runtime에서 사용하는 Artifact는 `git-pending`이나 `latest`가 아니라 검증된 Digest로 고정한다.

A-10 완료 시점의 확인된 Runtime 기준:

```text
Backend Digest
sha256:ad96e1515616c87540d73f1c3f64031afb7b4f9322a03a5a2759674d7da5b9c6

Frontend Digest
sha256:123203a4c5a5490690164deb68dffd929b6b9a45e36476e521f5c95ec70b0701
```

Evidence:

- `seokpan-app#78` — Backend `/metrics`를 포함한 최신 Backend Image Pipeline의 Test·Scan·Health Smoke·Digest 생성 근거
- `seokpan-app#3` — A-10 Runtime이 실제 소비한 Backend/Frontend Digest와 Deployment 상태를 연결한 Roadmap 근거

판정: `Validated`.

### 5.3 Backend / Frontend Desired State

Backend와 Frontend는 Argo CD Child Application으로 관리되며 실제 Runtime까지 활성화됐다.

현재 기준:

```text
Backend
- replicas: 2
- verified Harbor Digest
- worker-01 / worker-02 분산

Frontend
- replicas: 2
- verified Harbor Digest
- 동일 Origin Gateway 경로 사용
```

GitOps `Desired State`(Git에 선언된 Kubernetes 목표 상태), Argo CD Sync 상태, 실제 Pod Image Digest가 연결되어 있다.

판정: `Running / Validated for A-10`.

### 5.4 DB / Secret / TLS Provider

공식 DB Endpoint는 `db.seokpan.soldesk.store:3306`이다.

Runtime 계정:

```text
identity_svc
game_svc
```

Migration 전용 계정:

```text
db_admin
```

Kubernetes 공급 자산:

```text
application/backend-db-runtime
application/backend-db-migration
seokpan-internal-ca/ca.crt
/etc/seokpan/pki/ca.crt
```

실제 Runtime에서 다음을 확인했다.

- Backend Identity/Game Engine의 MaxScale TLS Session 성공
- TLS 1.3 연결
- Backend Pod가 최신 Root CA Fingerprint를 소비
- Runtime Credential과 Migration Credential 분리
- 70초 idle 이후 첫 후속 DB 요청과 Connection 교체 성공

Evidence:

- `seokpan-app#50` — Backend/Alembic이 MaxScale TLS를 통해 실제 DB Session을 생성하고 유지·재연결한 근거
- `seokpan-gitops#39` — Root CA 갱신 후 실행 중 Backend Pod의 CA Fingerprint와 MaxScale TLS 연결을 재확인한 근거

판정: `Validated`.

### 5.5 Migration

One-shot Migration은 일반 Backend Startup이나 Argo CD Auto-Sync가 자동 실행하지 않는다.

실제 Production DB에서 다음을 검증했다.

- 기존 Runtime DB Audit
- Alembic Baseline Stamp
- 후속 Revision 적용
- 기존 데이터 보존
- Replication 확인
- MaxScale Read/Write 확인
- 신규 빈 `stone_game` DB에서 `alembic upgrade head` 재현

Evidence:

- `seokpan-app#22` — 기존 Runtime DB와 신규 빈 DB 양쪽에서 수행한 Migration Gate 실행·검증 근거

```text
Migration Structure: Implemented / Merged
Static & Kubernetes API Validation: Validated
Actual Migration Gate: Validated
```

### 5.6 A-10 Production Provider Integration

A-10은 실제 MariaDB·Redis Provider와 Kubernetes/GitOps Runtime을 연결하는 단계이며 현재 완료됐다.

확인된 결과:

- Memory/Fake Provider 대신 실제 MariaDB·Redis를 사용하는 Production Provider 구성
- Backend 1 Replica Provider Gate PASS
- Backend 2 Replica 확장 및 Worker 분산
- Redis Session 공유 기본 경로 확인
- Frontend 2 Replica Runtime
- GitOps → Argo CD → Kubernetes 실제 CD 경로 PASS
- Argo CD Self-Heal PASS
- Gateway HTTPS / API / WebSocket Runtime Gate PASS
- Backend `/metrics` 제공
- ServiceMonitor 활성화 및 Prometheus Target 2개 `UP`
- 실제 Application Metric Query PASS

Evidence:

- `seokpan-app#3` — A-10 전체 완료 경계와 P4 후속 경계를 관리하는 Roadmap
- `seokpan-gitops#57` — Manual GitOps → Argo CD → Kubernetes CD와 Self-Heal Runtime 검증
- `seokpan-gitops#91` — Backend `/metrics`, ServiceMonitor, Prometheus Target과 실제 Metric Query 검증

판정:

```text
A-10 Runtime Integration: Completed
P4 Stabilization / Acceptance: In Progress
```

P4에서 확인 중인 Game/Realtime/2 Replica Edge Case를 A-10 미완료로 되돌려 표현하지 않으며, 반대로 A-10 완료를 최종 MVP Acceptance 완료로 확대하지 않는다.

## 6. Provider / Consumer 연결 기준

### Database

```text
MariaDB / MaxScale
→ db.seokpan.soldesk.store:3306
→ backend-db-runtime
→ identity_svc / game_svc
→ Backend
```

Migration:

```text
MariaDB / MaxScale
→ backend-db-migration
→ db_admin
→ One-shot Migration Job
```

`Secret Provider Ready ≠ Migration 성공 ≠ Backend DB 연결 성공`

### Redis

```text
Redis StatefulSet
→ redis.platform.svc.cluster.local:6379
→ SEOKPAN_REDIS_URL
→ Backend Production Provider
```

`Redis Running ≠ Backend 연결 ≠ Session/Room/Vote Provider 검증`

### Image / GitOps

```text
Application Source
→ Jenkins / BuildKit
→ Harbor / Scan / Digest
→ GitOps Desired State
→ PR / Merge
→ Argo CD
→ Kubernetes
```

과거 Commit의 검증 Image는 Source 변경 후 현재 Release Artifact로 자동 재사용하지 않는다.

### Gateway

```text
Common VIP
→ HAProxy
→ Gateway
→ HTTPRoute
→ Frontend / Backend Service
```

Gateway Platform과 Application Route 검증을 분리한다.

### Observability

Observability Platform 자체는 Delivery/Observability 담당 영역이다. 본 역할은 Application이 Health·Metric·Log·Replica/Pod 상태를 노출하고, Observability가 이를 수집·조회할 수 있도록 필요한 연결 지점과 확인 기준을 제공한다.

`Observability Platform Running ≠ Application Observability Integration Validated`

세부 Query·Dashboard·Alert·Evidence는 12에서 다룬다.

## 7. Integration 흐름

A-10에서 실제로 통과한 Runtime Integration 흐름은 다음과 같다.

```text
seokpan-app Source
→ Jenkins Image Pipeline
→ Harbor Verified Digest
→ seokpan-gitops Desired State
→ 승인된 Migration Gate
→ Backend 1 Replica
→ MariaDB / Redis / CA / Secret Consumer 검증
→ Backend 2 Replica / Worker 분산
→ Frontend 2 Replica
→ HTTPRoute
→ Gateway
→ HTTPS / API / WebSocket
→ Browser 접속
→ Application Metrics / Prometheus
```

여기까지가 A-10 Runtime Integration 완료 범위다.

이후 흐름은 다음과 같이 분리한다.

```text
A-10 Completed
→ P4 Stabilization
→ Concurrency / Realtime / Recovery / Performance
→ Final MVP Acceptance
```

각 구현 자산은 해당 Repository가 `Source of Truth`(그 자산의 현재 값을 최종적으로 관리하는 저장소)이며, 09는 역할·Dependency·Gate 상태를 연결한다.

## 8. Integration Gate

### Gate A — Base Platform Ready

Control Plane 3대, Worker 2대, Calico, CoreDNS, Namespace/RBAC, Argo CD, Gateway Platform, Redis Runtime을 확인했다.

현재: `PASS`.

### Gate B — Build & Artifact Ready

Jenkins Pipeline과 Harbor Artifact 경로가 현재 A-10 Source 기준 Image까지 검증됐다.

현재: `PASS`.

Evidence:

- `seokpan-app#78` — 최신 Backend Image Pipeline의 Scan·Health Smoke·Digest 생성 근거
- `seokpan-app#3` — 실제 Runtime이 소비한 Backend/Frontend Digest 연결 근거

### Gate C — Provider Ready

MariaDB, MaxScale, TLS/CA, DB Runtime/Migration Secret, Redis Runtime, One-shot Migration 실행 자산을 확인했다.

현재: `PASS`.

### Gate D — Backend First Runtime

실제 순서:

```text
Approved Migration
→ Backend replicas: 1
→ Pod Running / Ready
→ MaxScale TLS
→ Identity / Game DB
→ Redis
→ Provider-aware Readiness
```

현재: `PASS`.

Evidence:

- `seokpan-app#22` — Production Migration Gate
- `seokpan-app#50` — 실제 MaxScale TLS Session
- `seokpan-gitops` PR #93 — Backend 1 Replica Runtime 활성화와 Provider Gate

### Gate E — Scale-out / Frontend

확인된 범위:

- Backend 1 → 2 Replica
- worker-01 / worker-02 분산
- PDB / RollingUpdate
- Redis Session 공유 기본 경로
- Frontend 2 Replica Runtime

현재:

```text
2-Replica Scale-out: PASS
Frontend Runtime: PASS
A-10 Gate E: PASS

HPA Validation Track: Not Implemented / Not Tested
```

HPA는 A-10 Gate E 완료와 분리된 후속 Validation Track이다.

HPA 미구현은 Frontend / External Runtime 진행이나 A-10 완료를 취소하지 않으며, 구현·측정하지 않은 HPA 결과를 PASS로 기록하지 않는다.

Cross-Pod WebSocket replacement, Chat 권한, Runner/Realtime Edge Case는 P4 Stabilization에서 별도로 검증한다.

### Gate F — External Route / Transport Integration

확인된 범위:

- Frontend Route
- Backend `/api/v1`
- Backend `/ws/v1`
- Gateway
- HTTPS
- WebSocket
- Same-Origin 경로
- Let's Encrypt Production TLS
- Windows Host / Linux VM 실제 접속

현재: `PASS for A-10 external Runtime Gate`.

Evidence:

- `seokpan-infra#188` — Production Certificate, Gateway 적용, Windows/Linux 실제 접속
- `seokpan-app#3` — Gateway HTTPS/API/WebSocket Runtime Gate 완료

이 Gate의 PASS는 전체 게임 시나리오의 P4 Acceptance PASS와 동일하지 않다.

### Gate G — MVP Acceptance

A-10까지의 Runtime Integration은 완료됐지만, 최종 MVP Acceptance는 아직 진행 중이다.

현재:

```text
Application Metrics Integration: PASS
Argo CD Self-Heal: PASS
P4 Concurrency / Realtime / Recovery / Performance: In Progress / Not Yet Passed
Final MVP Acceptance: In Progress
```

현재 `seokpan-app#76`에서 서비스 안정화 항목을 추적한다.

버그픽스 중간 상태는 09에 세부 복제하지 않고 최종 검증 결과만 후속 현행화한다.

## 9. Gate 요약

| Gate | 현재 상태 | 의미 |
| --- | --- | --- |
| A Base Platform | PASS | Kubernetes·Network·Argo CD·Redis 등 A-10 기반 준비 완료 |
| B Build & Artifact | PASS | 현재 Runtime이 소비하는 검증 Image/Digest 확보 |
| C Provider | PASS | DB·Redis·Secret·CA·Migration 실행 기반 준비 및 연결 |
| D Backend First Runtime | PASS | Migration 후 Backend 1 Replica 실제 Provider 연결 검증 |
| E Scale-out / Frontend | PASS | Backend 2 Replica·Worker 분산·Frontend 2 Replica PASS. HPA는 별도 미검증 Track |
| F External Route / Transport | PASS | HTTPRoute·HTTPS·API·WebSocket·Windows/Linux 접속 Runtime Gate PASS. Realtime semantics 전체는 P4 별도 검증 |
| G MVP Acceptance | In Progress | A-10 완료 이후 P4 Stabilization / Acceptance 진행 중 |

A-10 완료 판정과 Gate G 최종 Acceptance 판정을 분리한다.

## 10. 남은 Blocker와 Gap

A-10의 기존 직접 Blocker는 해소됐다.

현재 남은 작업은 **A-10 실행 자체의 Blocker가 아니라 P4 Stabilization / Acceptance의 Validation Gap**이다.

주요 P4 범위:

- Game Start/End 일관성
- Realtime lifecycle
- Safe Leave / Reconnect
- 2 Replica cross-Pod WebSocket/Chat 조합
- Room 참여·재시도 일관성
- 동시성 정확성
- 대표 부하 성능 측정
- Backend 장애 복구 측정
- 최종 Browser Acceptance

별도 미구현/미검증 항목:

- HPA
- Git Revert 기반 실제 Rollback Run
- Cross-role DR 이후 Application 정상화 Test
- 일부 Application Log 최종 Evidence

현재 진행 중인 버그픽스의 개별 식별 번호나 임시 Root Cause 가설은 이 문서에 중복 기록하지 않고 `seokpan-app#76`을 Source로 사용한다.

### Deferred / Non-blocking

- 추가 LB/MaxScale HA
- ANALYSIS Runtime
- Redis Sentinel / Cluster
- 2차 Hybrid Cloud 이전
- 현재 MVP Acceptance와 직접 관련 없는 추가 운영 기능

## 11. Critical Path

A-10 Critical Path는 완료됐다.

현재 Critical Path는 다음과 같다.

```text
A-10 Runtime Integration Completed
→ P4 기능 안정화
→ 핵심 Game / Realtime / 2 Replica 경로 재검증
→ Concurrency / Recovery / Performance Evidence
→ Final Browser Acceptance
→ MVP Acceptance
```

UX/UI 개선은 기능 안정화 결과를 가리지 않는 범위에서 별도로 진행하며, 진행 중인 버그픽스 중간 결과를 09 Current State로 확정하지 않는다.

## 12. 완료 기준

### 문서 완료

- 책임 경계가 명확함
- Current State와 계획이 구분됨
- Provider/Consumer 관계가 실제 Runtime 연결 상태와 함께 설명됨
- A-10 Runtime Integration과 P4 Stabilization / Acceptance가 구분됨
- Backend Scale-out 완료와 HPA 미구현 상태가 구분됨
- Gate A~G와 현재 상태가 실제 구현 근거와 일치함
- A-10 Blocker와 P4 Validation Gap이 구분됨
- Delivery / Observability Cross-role 경계가 정의됨
- 10·11·12와 중복이 최소화됨
- Baseline → Change → Repository → 09 → 11/12 Traceability가 연결됨

### A-10 Runtime Integration 완료 범위

```text
Base Platform
→ Current Artifact
→ Provider
→ Migration
→ Backend 1 Replica
→ Backend 2 Replica / Worker Distribution
→ Frontend 2 Replica
→ Application HTTPRoute
→ HTTPS / API / WebSocket
→ Application Metrics
```

위 범위는 실제 Runtime Evidence로 검증됐다.

### 아직 완료로 보지 않는 범위

```text
HPA
P4 Concurrency
P4 Performance
P4 Recovery
Git Revert Rollback Run
Cross-role DR 이후 Application 정상화
Final MVP Acceptance
```

A-10 완료를 위 미검증 항목의 완료로 확대하지 않는다.

상세 실행 절차는 11, Test Case·측정·PASS/FAIL Evidence는 12에서 관리한다.

## 13. Traceability

```text
03 역할·책임
→ 04 기술·논리 아키텍처
→ 05 물리 배치·Worker·Scale-out
→ 06 자동화·테스트 설계
→ 07 MVP·Provider/Consumer·Replica
→ 08 최종 기획안
→ PROJECT_CHANGES
→ MVP_IMPLEMENTATION_BASELINE
→ seokpan-infra / seokpan-gitops / seokpan-app
→ 09 실행·통합 실시설계
→ 11 Runbook
→ 12 Validation
```

현재 A-10 완료 상태를 확인하는 주요 Evidence 연결:

- `seokpan-app#3` — A-01~A-10 Roadmap과 A-10 완료 / P4 후속 경계
- `seokpan-app#22` — MariaDB Migration Gate, 기존 Runtime DB와 신규 빈 DB 검증
- `seokpan-app#50` — Backend/Alembic MaxScale TLS 실제 연결
- `seokpan-app#78` — Backend `/metrics` 구현과 검증 Image 생성
- `seokpan-gitops#7` — Redis Runtime/Persistence와 실제 Backend Consumer 연결
- `seokpan-gitops#57` — GitOps → Argo CD → Kubernetes CD 및 Self-Heal 검증
- GitOps PR #93 — Backend 1 Replica Provider Runtime Gate
- GitOps PR #94/#95 — Backend 2 Replica, Worker 분산, PDB, RollingUpdate
- GitOps PR #96 — Frontend 2 Replica와 Same-Origin Gateway 경로
- `seokpan-gitops#91` — Backend ServiceMonitor와 Prometheus Target/Metric Query
- `seokpan-infra#188` — Let's Encrypt Production TLS와 Windows Host/Linux VM 외부 접속

현재 진행 중인 P4 서비스 안정화의 상세 상태는 `seokpan-app#76`에서 추적한다.

Historical 문서는 당시 상태를 유지한다. 이후 상태는 Change → Implementation → Runtime Validation으로 연결하고, 09에 개별 실행 로그나 버그픽스 중간 결과를 복제하지 않는다.
