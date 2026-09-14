# MVP 실행·통합 실시설계
## Kubernetes & Application Integration

## 1. 목적과 범위

이 문서는 「石나가는 판단」 1차 프로젝트에서 Kubernetes와 Application Integration 영역의 책임, Current State, Provider/Consumer 관계, 선행조건, Integration Gate와 완료 경계를 정의한다.

01~08은 Historical Baseline이며, 본 문서는 이를 다시 서술하지 않는다. Baseline 이후 확정 변경은 `PROJECT_CHANGES.md`, 공용 구현 기준은 `MVP_IMPLEMENTATION_BASELINE.md`, 실제 상태는 `seokpan-app`·`seokpan-gitops`·`seokpan-infra`의 현재 `main`, Issue, PR, Runtime Evidence를 우선한다.

- 상세 실행 명령·적용·복구·Rollback 절차: 11 구축·자동화 Runbook
- 공식 Test Case·측정·PASS/FAIL·Evidence 상세: 12 검증·측정 계획
- Repository·Directory·Issue·Branch·PR·Review Governance: [`10_GitHub_협업_및_Repository_운영.md`](../10_GitHub_협업_및_Repository_운영.md)

## 2. 선행 기준과 상태 판정

직접 상위 기준은 다음 문서다.

- [`03_SeokPan_논리_역할_및_서비스_목록.md`](../03_SeokPan_논리_역할_및_서비스_목록.md)
- [`04_SeokPan_기술_비교_및_논리_아키텍처.md`](../04_SeokPan_기술_비교_및_논리_아키텍처.md)
- [`05_SeokPan_물리_아키텍처.md`](../05_SeokPan_물리_아키텍처.md)
- [`06_SeokPan_Ansible_자동화_테스트_설계.md`](../06_SeokPan_Ansible_자동화_테스트_설계.md)
- [`07_SeokPan_확장_호환형_MVP_도출.md`](../07_SeokPan_확장_호환형_MVP_도출.md)
- [`08_SeokPan_1차_프로젝트_기획안.md`](../08_SeokPan_1차_프로젝트_기획안.md)

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
- Application Runtime과 Observability 사이의 Health·Metric·Log 연결 계약

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

- Control Plane 3 / Worker 2
- kubeadm / Calico
- CP·Worker Ready
- kube-apiserver·CoreDNS·Calico 정상
- 기존 Runtime 기준 Kubernetes 회귀검증 완료

판정: `Validated`

현재 동작 중인 Cluster와 완전히 빈 환경에서의 전체 Clean Rebuild는 별도 상태로 본다.

### 4.2 Network / DNS

주요 Endpoint:

```text
harbor.seokpan.soldesk.store
db.seokpan.soldesk.store:3306
game.seokpan.soldesk.store
redis.platform.svc.cluster.local:6379
```

Endpoint/DNS 기반은 사용 가능한 상태다. 실제 Consumer 연결은 각 Gate에서 별도 판정한다.

### 4.3 Namespace / RBAC / Argo CD

주요 Namespace는 `application`, `platform`, `cicd`, `observability`, `storage-infra`다. Namespace/RBAC는 Argo CD 관리 경로에 편입되어 있고 역할별 권한 경계가 검증되었다.

Argo CD는 Gateway, Redis, Namespace/RBAC, Jenkins, Observability, Storage, Backend, Frontend Desired State를 관리한다.

판정: Platform `Validated`, Backend/Frontend Runtime 활성화는 별도 Gate.

### 4.4 Gateway

GatewayClass, Gateway, HTTP/HTTPS Listener, TLS Secret, Worker NodePort, Common VIP `10.1.93.90:443`, TLS Handshake와 `game.seokpan.soldesk.store` Hostname 경로까지 Platform 범위가 검증됐다.

실제 Application HTTPRoute는 아직 없다.

```text
Gateway Platform: Validated
Application Route: Not Implemented / Not Tested
HTTPS/WSS Application E2E: Not Tested
```

### 4.5 Redis

서비스용 Redis는 StatefulSet 1 Replica, Service 6379, NFS PVC, AOF, `appendfsync everysec` 기준이며 Write/Read와 Pod 재생성 후 데이터 유지가 검증됐다.

```text
Redis Platform Runtime: Validated
Backend → Redis Consumer: Not Tested
```

### 4.6 Autoscaling / Workload Distribution

Baseline은 Worker 2대를 Replica·HPA 검증축으로 유지한다. 검토 대상은 Metrics Server 기반 Resource Metrics, CPU·Memory 기반 HPA, Resource Request/Limit, Worker 분산, 필요 시 anti-affinity/topology spread와 PDB다.

현재 Backend는 `replicas: 0`이고 HPA에 필요한 Application Resource 기준과 HPA Desired State는 완료된 상태로 확인되지 않는다.

```text
Manual Runtime Scale-out: Not Tested
HPA Integration: Not Implemented / Not Tested
Workload Distribution Validation: Not Tested
```

이는 Backend 1 Replica의 선행조건이 아니라 이후 Scale-out Gate의 범위다.

## 5. Application Integration Current State

### 5.1 Application Roadmap

```text
A-01~A-08: 기능·Headless·Frontend First Success
A-09: Container / Jenkins / Image Acceptance — Completed
A-10: 실제 Provider / GitOps / Kubernetes Runtime Integration — In Progress
```

A-09 완료를 실제 MariaDB·Redis·Kubernetes Integration 완료로 확대하지 않는다.

### 5.2 Build / Artifact

A-09에서 Application 검사, Container Build, Jenkins, Rootless BuildKit, Harbor, Scan, Process/Health Smoke, SBOM, Provenance, Final Tag/Digest와 실패 시 Promote 차단 경로가 검증되었다.

```text
Delivery / Image Pipeline Capability: Validated
A-09 Accepted Artifact: Validated
Current A-10 Release Artifact: Pending
```

A-10 Source 변경 후 실제 Runtime이 소비할 현재 Release Image는 새 `main` 기준으로 다시 생성·검증해야 한다.

### 5.3 Backend / Frontend Desired State

Backend와 Frontend Desired State와 Argo CD Child Application은 존재한다.

현재 Backend:

```text
replicas: 0
image: git-pending
```

판정: Desired State Structure `Merged`, Runtime `Not Running`.

### 5.4 DB / Secret / TLS Provider

공식 DB Endpoint는 `db.seokpan.soldesk.store:3306`이다.

- Runtime: `identity_svc`, `game_svc`
- Migration: `db_admin`
- `application/backend-db-runtime`
  - `SEOKPAN_IDENTITY_DATABASE_URL`
  - `SEOKPAN_GAME_DATABASE_URL`
- `application/backend-db-migration`
  - `SEOKPAN_MIGRATION_DATABASE_URL`
- Root CA: `seokpan-internal-ca/ca.crt`
- Mount: `/etc/seokpan/pki/ca.crt`

Secret Provider는 검증됐고 MaxScale TLS Provider도 준비됐다. 실제 Backend DB TLS Session은 아직 미검증이다.

### 5.5 Migration

One-shot Migration 실행 자산은 `main`에 반영되어 있다.

- `batch/v1 Job`
- 실행별 고유 Job
- `backoffLimit: 0`
- 검증된 Backend Image Digest만 허용
- Migration 전용 Secret 사용
- Runtime Credential과 분리
- Root CA 재사용
- Mutation Approval Reference
- 일반 Backend Kustomization/Argo CD Auto-Sync와 분리

Renderer 정상·거부 경로와 Kubernetes API Server CREATE dry-run까지 검증됐다.

```text
Migration Structure: Implemented / Merged
Static & Kubernetes API Validation: Validated
Actual Migration: Not Tested
```

### 5.6 A-10 Production Provider Integration

A-10은 실제 MariaDB·Redis Provider를 사용하는 Production Runtime 조립 단계다. 09는 내부 App 구현 상세를 다시 소유하지 않고 다음 Integration 결과만 추적한다.

- Memory/Fake Provider 없이 Production Runtime 조립 가능
- MariaDB/Redis Runtime Adapter 연결 가능
- Provider Readiness 판정 가능
- 다중 Replica에서 공유 상태가 Process Memory에 종속되지 않음
- Kubernetes 종료·재기동 경계를 소비할 수 있음

판정: `In Progress`.

## 6. Provider / Consumer Contract

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

Observability Platform은 Delivery/Observability 영역이 소유한다. 본 역할은 Application Runtime에서 Health, Metric, Log, Replica/Pod 상태와 장애·재기동 식별이 연결될 수 있도록 Consumer 경계를 제공한다.

`Observability Platform Running ≠ Application Observability Integration Validated`

세부 Query·Dashboard·Alert·Evidence는 12에서 다룬다.

## 7. Integration 흐름

```text
seokpan-app A-10
→ 현재 main Image
→ Harbor Digest
→ seokpan-gitops Image Desired State
→ Migration Gate
→ Backend replicas: 1
→ DB / Redis / CA / Secret
→ Backend First Runtime
→ 2 Replica / Shared Runtime
→ HPA / Workload Distribution
→ Frontend Runtime
→ HTTPRoute
→ Gateway
→ HTTPS / WSS
→ Browser
```

각 구현 자산은 해당 Repository가 소유하며 09는 Integration 계약과 Gate를 관리한다.

## 8. Integration Gate

### Gate A — Base Platform Ready

Backend First Runtime에 필요한 CP3/Worker2, Calico, CoreDNS, Namespace/RBAC, Argo CD, Gateway Platform, Redis Platform Runtime을 확인한다.

현재: `PASS for First Runtime prerequisite`.

HPA용 Resource Metrics는 Gate E에서 별도로 다룬다.

### Gate B — Build & Artifact Ready

현재:

```text
Pipeline Capability: PASS
A-09 Accepted Artifact: PASS
Current A-10 Release Artifact: Pending
```

A-10 Source 변경 후 현재 Release Commit 기준 Image가 필요하다.

### Gate C — Provider Ready

MariaDB, MaxScale, TLS/CA, DB Runtime/Migration Secret, Redis Runtime, One-shot Migration 실행 자산을 확인한다.

현재: `Provider Prerequisite: PASS`.

실제 Migration과 Backend Consumer Session은 Gate D다.

### Gate D — Backend First Runtime

선행조건:

- A-10 Production Provider Integration 준비
- 현재 Release Backend Image
- 승인된 Migration 실행 조건
- Provider 상태
- Secret/CA
- GitOps Image Desired State

완료 경계:

```text
Actual Migration
→ Backend replicas: 1
→ Pod Running
→ Startup / Live / Readiness
→ MariaDB TLS Consumer
→ Redis Consumer
→ 기본 서비스 흐름
```

현재: `In Progress / Not Yet Passed`.

### Gate E — Scale-out / HPA / Frontend Integration

Gate D 이후 다음을 연결한다.

- Backend 1 → 2 Replica
- Redis 공유 Runtime State
- Replica 간 Realtime Event
- Runner 중복 실행 방지
- 재접속 상태 수렴
- Worker 간 Workload 분산
- Application Resource 기준
- Resource Metrics
- HPA Desired State와 Scale 동작
- Frontend Image / Deployment / Service

현재:

```text
2-Replica Integration: Not Tested
HPA: Not Implemented / Not Tested
Frontend Runtime Integration: Not Tested
```

HPA 임계값·대표 부하·Scale 시간·응답시간 측정은 12가 소유한다.

### Gate F — Realtime / External Integration

- Frontend Route
- Backend `/api/v1`
- Backend `/ws/v1`
- Gateway
- HTTPS/WSS
- Forwarded Header
- Cookie/CORS
- Common VIP
- Browser

현재: Gateway Platform은 이미 검증됐지만 Application Route/HTTPS/WSS/Browser는 `Not Tested`.

### Gate G — MVP Acceptance

실제 환경에서 기능·Runtime·Delivery·Observability·복구·운영 관점의 최종 완료를 판정한다.

```text
Base Platform
→ Current Artifact
→ Provider
→ Backend Runtime
→ Scale / HPA / Frontend
→ External Integration
→ Delivery / Observability Integration
→ MVP Acceptance
```

M5 First Success는 M3 Runtime과 M4 Delivery/Observability 선행조건을 충족한 뒤 수행한다. 이후 장애·복구·대표 부하를 포함한 MVP P4를 연결한다.

현재: `Not Tested`.

## 9. Gate 요약

| Gate | 현재 상태 | 의미 |
| --- | --- | --- |
| A Base Platform | PASS | Backend First Runtime 기본 Platform 준비 |
| B Build & Artifact | Partial | Pipeline/A-09 PASS, A-10 현재 Artifact 대기 |
| C Provider | PASS | DB·Redis·Secret·CA·Migration 실행 자산 준비 |
| D Backend First Runtime | In Progress | A-10·실제 Migration·1 Replica 대기 |
| E Scale/HPA/Frontend | Not Tested | 2 Replica·HPA·Frontend 대기 |
| F Realtime/External | Not Tested | Route·HTTPS/WSS·Browser 대기 |
| G MVP Acceptance | Not Tested | Runtime·Delivery/Observability·P4 대기 |

## 10. 남은 Blocker와 Gap

### Gate D 직접 선행사항

- A-10 Production Provider Integration 준비
- 현재 Backend Image 생성·검증
- 승인된 실제 Migration
- Backend GitOps Image 갱신
- `replicas: 1` 활성화

### 이후 Implementation / Validation Gap

- Backend → MariaDB TLS
- Backend → Redis
- Provider Readiness
- Backend 2 Replica
- 공유 Runtime State / Replica 간 Realtime Event
- HPA / Resource Metrics / Workload 분산
- Frontend Runtime
- HTTPRoute
- HTTPS/WSS Application 경로
- Browser E2E
- Application Observability Integration
- 최종 MVP P4

모든 Gap을 현재 Gate의 Blocker라고 표현하지 않는다.

### Deferred / Non-blocking

- 추가 LB/MaxScale HA
- ANALYSIS Runtime
- Redis Sentinel / Cluster
- 2차 Hybrid Cloud 이전
- 현재 MVP 검증축과 무관한 추가 운영 기능

## 11. Critical Path

```text
A-10 Provider Integration
→ 현재 Backend Image
→ Actual Migration
→ Backend 1 Replica
→ DB + Redis + Readiness
→ Backend First Runtime
→ Backend Scale-out
→ HPA / Workload Distribution
→ Frontend Runtime
→ HTTPRoute
→ HTTPS / WSS
→ Browser E2E
→ MVP Acceptance
```

독립적인 준비 작업은 병렬 진행할 수 있지만 자산의 존재를 실제 Consumer Integration 성공으로 대신하지 않는다.

## 12. 완료 기준

### 문서 완료

- 책임 경계가 명확함
- Current State와 계획이 구분됨
- Provider/Consumer 관계가 정의됨
- Platform과 Application Integration 완료가 구분됨
- Scale-out·HPA가 누락되지 않음
- Gate A~G와 현재 상태가 실제 구현 근거와 일치함
- Blocker와 Implementation/Validation Gap이 구분됨
- Delivery/Observability Cross-role 경계가 정의됨
- 10·11·12와 중복이 최소화됨
- Baseline → Change → Repository → 09 → 11/12 Traceability가 연결됨

### Runtime 완료

```text
Base Platform
→ Current Artifact
→ Provider
→ Migration
→ Backend 1 Replica
→ Scale-out / HPA
→ Frontend
→ Gateway Route
→ HTTPS / WSS
→ Delivery / Observability Integration
→ Browser
→ MVP Acceptance
```

상세 실행은 11, 공식 시험·측정 결과는 12에서 연결한다.

## 13. Traceability

```text
03 역할·책임
→ 04 기술·논리 아키텍처
→ 05 물리 배치·Worker·HPA
→ 06 자동화·테스트 설계
→ 07 MVP·Provider/Consumer·Replica/HPA
→ 08 최종 기획안
→ PROJECT_CHANGES
→ MVP_IMPLEMENTATION_BASELINE
→ seokpan-infra / seokpan-gitops / seokpan-app
→ 09 실행·통합 실시설계
→ 11 Runbook
→ 12 Validation
```

주요 구현 추적 대상:

- `seokpan-app#3`, A-09 관련 작업, A-10 Production Provider Integration
- `seokpan-app#22`, `seokpan-app#50`
- `seokpan-infra#150`
- `seokpan-gitops#7`, `#29`, `#40`, `#44`, PR #46
- Backend/Frontend Argo CD Child Application
- Gateway HTTPS/TLS Platform 및 Application HTTPRoute 후속 작업

Historical 문서는 당시 상태를 유지한다. 이후 상태는 Change → Implementation → Runtime Validation으로 연결하고, 09에 상세 실행 이력이나 개별 시험 결과를 누적하여 Runbook 또는 Validation 문서로 변질시키지 않는다.
