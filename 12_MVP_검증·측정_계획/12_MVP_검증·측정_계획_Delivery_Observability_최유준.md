# MVP 검증·측정 계획
## Delivery / Observability

## 1. 목적과 문서 경계

이 문서는 「石나가는 판단」 1차 프로젝트의 Delivery / Observability 영역에서 **무엇을 시험하고, 무엇을 측정하며, 어떤 근거로 PASS/FAIL을 판정할지** 정의하고, 1차 프로젝트 종료 시점(2026-09-23)의 실제 판정 결과를 기록한다.

```text
09 = What / Why / Responsibility / Gate(A~G)
11 = Pre-check / Apply / Verify / Re-run / Recovery / Rollback
12 = Test Case / Preconditions / Stimulus·Fault / Measurement / PASS·FAIL / Evidence
```

12는 11의 실행 명령을 복제하지 않는다. 실제 실행 절차는 11을 참조하고, 본 문서는 각 Test의 선행조건·실행 자극/장애·관찰 항목·PASS/FAIL 기준·측정 Evidence를 정의한다.

직접 기준:

- [`02_SeokPan_핵심_문제_및_검증_목표.md`](../01-08_기획·설계_Baseline/02_SeokPan_핵심_문제_및_검증_목표.md) — C-06 관측·검증성
- [`06_SeokPan_Ansible_자동화_테스트_설계.md`](../01-08_기획·설계_Baseline/06_SeokPan_Ansible_자동화_테스트_설계.md) — G-06, G-07, F-10~F-13, Q-07, REQ-CICD, REQ-OBS
- [`09_MVP_실행·통합_실시설계_Delivery_Observability_최유준.md`](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Delivery_Observability_최유준.md)
- [`11_MVP_구축·자동화_Runbook_Delivery_Observability_최유준.md`](../11_MVP_구축·자동화_Runbook/11_MVP_구축·자동화_Runbook_Delivery_Observability_최유준.md)
- [`12_MVP_검증·측정_계획_Kubernetes_Application_Integration_정태훈.md`](12_MVP_검증·측정_계획_Kubernetes_Application_Integration_정태훈.md) — KAI-DEL-01 / KAI-OBS-01·02 / KAI-CD-01·02 Cross-role
- [`PROJECT_CHANGES.md`](../PROJECT_CHANGES.md)
- `seokpan-gitops`, `seokpan-infra`, `seokpan-app` 최신 `main`, Issue, PR, Runtime Evidence

실제 Test Run 직전에는 관련 Repository의 최신 Revision과 열린 Issue/PR을 다시 확인한다.

---

## 2. 상위 검증축과 Gate 대응

02의 검증축을 대체하지 않는다. 본 역할은 M-01~M-05 중 어느 하나의 결과 자체를 소유하지 않지만, **모든 검증축이 사용하는 Metric·Log·Timeline·Evidence 계층**(C-06)과 **배포 경로**를 제공한다.

| ID | 검증축 | 본 역할과의 관계 |
| --- | --- | --- |
| M-01 | 동시성 정확성 | Cross-role — 409 충돌 거부 실시간 대리 신호(Service / KPI Dashboard), 최종 판정은 DB 검증 |
| M-02 | Vote 처리 성능 | Cross-role — 처리량·p95/p99·5xx 오류율 Metric과 부하 시 자원 사용 제공 |
| M-03 | Backend 장애 복구 | Cross-role — 장애 시각·Pod 재시작·Log Timeline 제공 |
| M-04 | DR | Cross-role — MariaDB/외부 VM Metric, Alert 경로 제공 |
| M-05 | Ansible 개선 | 핵심 — 본 영역 Role(harbor, jenkins_secrets, argocd_bootstrap, alertmanager_*, 외부 Exporter)의 재실행·멱등성 |
| C-06 | 관측·검증성 | **핵심** — Metric·Log·Alert·Timeline 수집 가능 여부 |

06 Gate와의 대응:

```text
G-06 Delivery       (Harbor, Jenkins Agent, GitOps Sync / Commit→Digest→Healthy)
  → DOB-REG-*, DOB-CI-*, DOB-CD-*
G-07 Observability  (Scrape, Log, Alert firing/resolved / Metric·Log 조회와 E-mail)
  → DOB-MET-*, DOB-LOG-*, DOB-ALT-*, DOB-DSH-*
F-10~F-13           → DOB-FLT-01~04
Q-07 CI/CD          → DOB-Q07-01
```

09 Gate와의 대응:

```text
Gate A (Registry Ready)              → DOB-REG-01 / DOB-REG-02
Gate B (CI Build & Artifact Ready)   → DOB-CI-01 / DOB-CI-02 / DOB-CI-03
Gate C (GitOps CD Ready)             → DOB-CD-01 / DOB-CD-02 / DOB-CD-03
Gate D (Delivery Automation)         → DOB-CD-04 / DOB-CD-05
Gate E (Metric Collection)           → DOB-MET-01 / DOB-MET-02 / DOB-MET-03
Gate F (Log Collection)              → DOB-LOG-01 / DOB-LOG-02 / DOB-LOG-03
Gate G (Alert / Dashboard)           → DOB-ALT-01 / DOB-ALT-02 / DOB-ALT-03 / DOB-DSH-01 / DOB-DSH-02
```

---

## 3. 상태와 측정 원칙

### 3.1 상태

```text
Defined
Implemented
Merged
Running
Validated
Partial
Blocked
Deferred
Not Implemented
Not Tested
Failed
```

다음을 같은 의미로 취급하지 않는다.

```text
Image Pushed ≠ Scan PASS ≠ Final Digest ≠ GitOps PR ≠ Merged ≠ Argo Synced ≠ Pod Ready(해당 Digest)
PR 생성 자동화 ≠ Merge 자동화 (Merge는 의도적으로 사람이 수행)
Stack Running ≠ Target UP ≠ Query 결과 존재 ≠ Dashboard 표시
Alert firing(API) ≠ E-mail 수신 ≠ resolved 수신
Loki query_range status:success ≠ 실제 Log Stream 반환
```

### 3.2 임의 목표값 금지

다음은 실험·실측 근거 없이 확정하지 않는다.

- Pipeline 소요 시간 목표
- Commit → Ready 목표 시간
- Self-Heal / Rollback 복구 시간 목표
- Alert 통보 지연 허용값
- Alert Rule 임계값(복제 지연 초, 오류율 % 등)
- Dashboard 색상 임계값
- Prometheus/Loki 보존 기간 변경

이미 측정된 값(7장)은 **해당 Run의 관찰값**이며 목표값이 아니다. 조건(Pipeline 구성, Review 대기, 클러스터 부하)이 바뀌면 새 Run으로 재측정하고 이전 값을 덮어쓰지 않는다.

### 3.3 현재 MVP 범위

1차 프로젝트 MVP 범위:

- Delivery: Harbor → Jenkins → GitOps PR → Argo CD → Kubernetes E2E, Self-Heal, Git Revert Rollback
- Observability: In-cluster / 외부 VM Metric, In-cluster Pod Log·Node Journal, Alert E-mail firing/resolved, Provisioning Dashboard

MVP 범위 밖(Deferred, 완료 조건에 포함하지 않음):

- 외부 VM Log 수집(`alloy_linux`)
- 프로젝트 전용 PrometheusRule(복제 중단·지연, M-축 KPI) — PROJECT_CHANGES 2026-09-21 2차 이관
- 장애 주입 시험 F-10~F-13
- Q-07 수동 대비 CI/CD 시간 비교
- Harbor Retention, Jenkins Job Git 선언
- Observability 도구 자체 HA·장기 저장·Discord(07 문서에서 이미 후속으로 분리)

---

## 4. Evidence 규격

각 Run 최소 Metadata:

```text
Test Case ID
Run ID
Target — harbor / jenkins / pipeline / promotion / argocd / prometheus / loki / alertmanager / grafana / exporter
Start / End Timestamp (KST, 동일 기준시계)
Operator
Repository Commit SHA (app / gitops / infra 중 해당)
Jenkins Build Number (Delivery Test)
Argo CD Revision (CD Test)
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
image-metadata.json / gitops-promotion-result.json
Harbor Artifact Digest 조회 결과
Argo CD Application status (sync / health / revision)
Pod imageID
Prometheus /api/v1/targets 요약 (job × health)
PromQL / LogQL Query와 결과
Alertmanager /api/v2/status, /api/v2/alerts
수신 E-mail 헤더(Authentication-Results: spf / dkim / dmarc) — 본문·주소 외 개인정보 제외
Ansible 실행 요약 (ok / changed / failed)
```

금지:

```text
Robot Password / Harbor Admin Password
GitHub PAT / Webhook Secret
Resend API Key / SMTP Password
Vault 평문 값
kubeconfig Credential
Jenkins Admin Password
Kubernetes Secret 실제 값
```

### 4.1 Run ID

```text
<TEST-CASE-ID>-YYYYMMDD-HHMM-<short-sha>
```

예:

```text
DOB-CD-04-20260922-1918-7746f71
```

### 4.2 실패 Run 보존

```text
실패 Run 보존
→ 원인·조치 기록
→ 새 Run ID
→ 재실행
```

FAIL을 수정해서 PASS로 덮어쓰지 않는다. 예: TS-035의 Plugin Lock 1차 배포 실패(GitOps PR #62 → Revert #64)는 삭제하지 않고, 수정된 PR #65의 성공 Run과 함께 보존한다.

---

## 5. 정량 측정 Catalog

| 항목 | 단위 | 측정 방법/출처 | 용도 |
| --- | --- | --- | --- |
| main Pipeline 소요 시간 | s | Jenkins Build duration | Q-07, M-05 |
| Stage별 소요 시간(Build/Scan/Smoke/Promote/Promotion) | s | Jenkins Stage View / timestamps | 병목 진단 |
| Scan 결과 | CRITICAL / fixable HIGH count | Trivy JSON | Gate B |
| App Merge → Promotion PR 생성 | s | App merge 시각 ↔ GitHub PR `created_at` | Delivery 자동화 지연 |
| Promotion PR 생성 → Merge | s | PR `created_at` ↔ `merged_at` | Review 대기(사람) |
| GitOps Merge → Argo Synced | s | Merge 시각 ↔ Argo `operationState.finishedAt` | Webhook 효과 |
| Argo Synced → Pod Ready | s | Argo ↔ Pod Ready condition | Rollout |
| Self-Heal 복귀 시간 | s | Drift 시각 ↔ Argo sync event | Gate C |
| Rollback 복구 시간 | s/min | 불량 Merge ↔ Revert Merge ↔ Healthy | Gate C |
| 가용 Replica 최저값(Rollback 중) | count | `kubectl get deploy -w` | 가용성 영향 |
| Prometheus Target UP 비율 | UP/total (job별) | `/api/v1/targets` | Gate E |
| Loki 수집 Node 수 | count | `count by (node_name)` LogQL | Gate F |
| Log ingestion delay | s | 기준 Log 발생 시각 ↔ Loki 조회 가능 시각 | Gate F |
| Alert firing → E-mail 수신 | s | Alertmanager `startsAt` ↔ 메일 수신 시각 | Gate G |
| resolved E-mail 수신 | s | Rule 제거 시각 ↔ 메일 수신 시각 | Gate G |
| E-mail 인증 결과 | pass/fail | SPF / DKIM / DMARC 헤더 | Gate G |
| Ansible 재실행 changed | count | 2회차 실행 결과 | M-05 |

측정 출처가 여러 개인 경우 Run Metadata에 실제 Source를 명시한다.

---

## 6. Dependency Evidence 재사용

Revision이 바뀌지 않은 검증 완료 Capability는 재사용할 수 있다.

- Harbor 설치·TLS·Robot·Immutability(Infra PR #157/#161 이후 변경 없음)
- Jenkins JCasC·Plugin Lock(GitOps PR #65) / Credential 경로(GitOps PR #127)
- Argo CD Bootstrap(Infra PR #209 이후)
- Observability NetworkPolicy(GitOps PR #112 이후)
- Alertmanager SMTP 구성(GitOps PR #97/#101/#129)

재검증 조건:

```text
harbor / jenkins_secrets / argocd_bootstrap / alertmanager_* Role 변경
Jenkins Controller / Plugin / Agent Image Digest 변경
Jenkinsfile.image-pipeline / promote_gitops.py 변경
kube-prometheus-stack Chart 버전 또는 values 변경
NetworkPolicy 변경 (observability 또는 인접 namespace)
Resend 도메인 / DNS 레코드 변경
Argo CD 버전 변경 또는 재설치
장애·복구 후
```

---

## 7. 현재 검증 기준 상태

| 영역 | 상태 | 12 판정 |
| --- | --- | --- |
| Harbor Registry / TLS / Robot(Gate A) | Validated | PASS |
| Tag Immutability + `scan-*` 예외(Gate A) | Validated | PASS |
| PR Pipeline(Gate B) | Validated | PASS |
| main Image Pipeline(Gate B) | Validated | PASS |
| Jenkins 재현성(Plugin Lock·Digest 고정)(Gate B) | Validated | PASS |
| Digest Pinning 배포(Gate C) | Validated | PASS(Cross-role, KAI-DEL-01) |
| Self-Heal(Gate C) | Validated | PASS(Cross-role, KAI-CD-01) |
| Git Revert Rollback(Gate C) | Validated | PASS — 가용성 1/2 저하 관찰 기록 |
| GitOps Promotion PR 자동화(Gate D) | Validated | PASS — 실제 PR 3건 |
| Argo CD Webhook(Gate D) | Validated | PASS(동작) / 지연 단축 수치 Not Measured |
| In-cluster Metric(Gate E) | Validated | PASS |
| 외부 VM Metric(Gate E) | Validated | PASS |
| Application Metric(Gate E) | Validated | PASS(Cross-role, KAI-OBS-01) |
| Pod Log / Node Journal(Gate F) | Validated | PASS |
| 외부 VM Log(Gate F) | Not Implemented | Not Tested / Deferred |
| Alert firing → resolved API(Gate G) | Validated | PASS |
| E-mail 수신(Gate G) | Validated | PASS |
| 프로젝트 전용 Alert Rule(Gate G) | Partial | Deferred |
| Dashboard Provisioning / 4종(Gate G) | Validated | PASS |
| F-10~F-13 장애 주입 | Not Tested | Not Tested |
| Q-07 CI/CD 비교 | Not Tested | Not Tested(Merge 시각 관찰값만 존재) |

### 7.1 관찰된 측정값 (해당 Run 기준, 목표값 아님)

| 항목 | 값 | 출처 |
| --- | --- | --- |
| App Merge → Promotion PR Merge(1회차, backend+frontend) | 12분(19:06 → 19:18) | App `b774a40`, GitOps #132 |
| App Merge → Promotion PR Merge(2회차, frontend) | 11분(20:10 → 20:21) | App `4b803c3`, GitOps #133 |
| App Merge → Promotion PR Merge(3회차, frontend) | 2시간 4분(22:16 → 00:20, Review 대기 포함) | App `b281976`, GitOps #134 |
| Self-Heal 복귀 | 수 초(scale 2→3 → 2) | seokpan-gitops#57 Step 7 |
| 불량 Digest Merge → Revert Merge | 11분(10:04 → 10:15, 원인 확인 포함) | GitOps #119 / #120 |
| Rollback 중 최저 가용 Replica | 1/2 | seokpan-gitops#57 Step 8 |
| Jenkins Controller 비가용(TS-035) | 약 9분(19:41 Merge → 19:50 Revert) | GitOps #62 / #64 |
| Prometheus Target Down 해소 | 19건 → 0(비활성 4종 제외) | Issue #69, GitOps PR #70/#71 |
| 외부 VM node_exporter | 7/7 UP | seokpan-gitops#103, Issue #205 |
| mariadb-exporter / harbor-metrics | 2/2 / 1/1 UP | GitOps PR #106 |
| Alloy DaemonSet / Journal 수집 Node | 5/5 / 5 | GitOps PR #81 / #99 |
| E-mail 인증 | SPF / DKIM / DMARC pass | GitOps PR #97 |

**주의**: Merge 시각 기반 값은 Jenkins Pipeline 소요와 사람의 Review 대기가 섞여 있다. 두 구간을 분리한 값(App Merge → PR `created_at`, PR `created_at` → `merged_at`)은 DOB-Q07-01에서 별도로 측정한다.

---

## 8. Test Traceability Matrix

| Test Case | 목적 | Gate | 11 | 현재 상태 |
| --- | --- | --- | --- | --- |
| DOB-PRE-01 | Dependency / Revision Snapshot | 공통 | 4 | Validated |
| DOB-REG-01 | Harbor Registry·TLS·Robot 분리 | A | 5.1 / 5.3 | PASS |
| DOB-REG-02 | Tag Immutability + `scan-*` 예외 | A | 5.2 | PASS |
| DOB-CI-01 | PR Pipeline | B | 7.1 | PASS |
| DOB-CI-02 | main Image Pipeline(Scan·Smoke·Digest·Evidence) | B | 7.2 | PASS |
| DOB-CI-03 | Jenkins 재현성(Plugin Lock·Digest·JCasC) | B | 6 | PASS |
| DOB-CD-01 | Digest Pinning 배포 → Pod ImageID | C | 8.3 | PASS(Cross-role KAI-DEL-01) |
| DOB-CD-02 | Self-Heal | C | 9.1 | PASS(Cross-role KAI-CD-01) |
| DOB-CD-03 | Git Revert Rollback | C | 9.2 | PASS |
| DOB-CD-04 | GitOps Promotion PR 자동화 | D | 8.1~8.3 | PASS |
| DOB-CD-05 | Argo CD Webhook | D | 8.6 | PASS(동작) / 지연 Not Measured |
| DOB-MET-01 | In-cluster Target | E | 10.3 | PASS |
| DOB-MET-02 | 외부 VM Exporter | E | 13 | PASS |
| DOB-MET-03 | Application Metric | E | 10.3 | PASS(Cross-role KAI-OBS-01) |
| DOB-LOG-01 | Pod Log | F | 11.3 | PASS |
| DOB-LOG-02 | Node Journal | F | 11.2~11.3 | PASS |
| DOB-LOG-03 | 외부 VM Log | F | — | Not Implemented / Deferred |
| DOB-ALT-01 | Alert firing → resolved(API) | G | 12.4 | PASS |
| DOB-ALT-02 | E-mail 수신·인증 | G | 12.1~12.4 | PASS |
| DOB-ALT-03 | 프로젝트 전용 Alert Rule | G | — | Deferred |
| DOB-DSH-01 | Dashboard / Datasource Provisioning | G | 14 | PASS |
| DOB-DSH-02 | Dashboard 관점 분리·내용 정합 | G | 14.1 | PASS |
| DOB-AUT-01 | 본 영역 Ansible Role 멱등성 | M-05 | 16.1 | Partial |
| DOB-FLT-01 | F-10 Harbor 중단 | — | — | Not Tested |
| DOB-FLT-02 | F-11 Jenkins 중단 | — | — | Not Tested |
| DOB-FLT-03 | F-12 Alertmanager 중단 | — | — | Not Tested |
| DOB-FLT-04 | F-13 Prometheus/Loki Node 유실 | — | — | Not Tested |
| DOB-Q07-01 | CI/CD 수동 대비 시간 | — | — | Not Tested |

`PASS`는 해당 Test Case의 현재 정의 범위에 실제 실측 Evidence가 존재할 때만 사용한다.

`DOB-CD-04`의 PASS는 "PR 생성까지 자동화"에 대한 판정이다. Merge 자동화는 설계상 존재하지 않으므로 판정 대상이 아니다.

---

## 9. Test Contract Matrix

| Test | Preconditions | Stimulus / Fault | Observation | PASS 핵심 | Evidence |
| --- | --- | --- | --- | --- | --- |
| DOB-PRE-01 | 대상 Revision 식별 | 없음, Snapshot | Argo App / Secret Key / Harbor health | Dependency와 Revision 일치 | Snapshot / Commit |
| DOB-REG-01 | Harbor 설치 | 재부팅 / Push / Pull | systemd, TLS 검증, Robot 권한 | 자동 기동, x509 없음, 권한 분리 | TS-010/017/024, Infra PR #161 |
| DOB-REG-02 | Immutability Rule | `git-*` 재Push·삭제, `scan-*` 생성·삭제 | HTTP 코드 | `git-*` 412 거부, `scan-*` 201/200 | TS-027/034 |
| DOB-CI-01 | PR 생성 | PR Pipeline 실행 | Stage 결과, Push 여부 | 검사·Build PASS, Push 없음 | Jenkins PR Build |
| DOB-CI-02 | main Merge | main Pipeline 실행 | Scan / Smoke / Promote / Digest / Evidence | 전 Stage PASS, Final Digest == Candidate | image-metadata.json |
| DOB-CI-03 | Plugin Lock, Digest 고정 | Controller 재생성 | Plugin 설치, Credential 등록 | 76 Plugin 설치, Credential ID 3종 | TS-035, GitOps PR #65/#127 |
| DOB-CD-01 | 검증 Digest | GitOps Digest 반영 | Argo revision, Pod imageID | Digest 일치, Ready | seokpan-gitops#57 |
| DOB-CD-02 | Synced 상태 | Live Drift(replicas) | OutOfSync → Synced | Git 상태로 복귀 | seokpan-gitops#57 Step 7 |
| DOB-CD-03 | 정상 Baseline Digest | 불량 Digest Merge → Revert | ImagePullBackOff → 복구, 가용 Replica | Baseline Digest 복귀, Healthy | GitOps #119/#120 |
| DOB-CD-04 | Promotion Credential, annotation | App main Merge | 결과 코드, PR 내용 | 영향 Component만 정확히 PR, auto-merge 없음 | App #103, GitOps #132~#134 |
| DOB-CD-05 | Webhook Secret → Route | GitOps push | Recent Deliveries, argocd-server log | 200 응답, Refresh 발생, 기존 Route 회귀 없음 | GitOps #123, Infra #210 |
| DOB-MET-01 | NetworkPolicy | Target 조회 | job × health | 비활성 4종 외 Down 0 | GitOps PR #70/#71 |
| DOB-MET-02 | Exporter·방화벽·EndpointSlice | Target 조회 + curl | job × health | node-exporter-external 7/7, mariadb 2/2, harbor 1/1, lb01 UP | #103, Infra #206/#208 |
| DOB-MET-03 | `/metrics`, label 정합 | ServiceMonitor 활성화 | Target, Query | Target UP, 실제 Metric 조회 | seokpan-gitops#91 |
| DOB-LOG-01 | Alloy DaemonSet | Pod Log 발생 | LogQL, label | Stream 반환, label 일치 | GitOps PR #81 |
| DOB-LOG-02 | SELinux Canary 통과 | Journal 수집 | Node별 count | 5 Node 연속 수집, Pod Log 회귀 없음 | GitOps PR #99 |
| DOB-LOG-03 | (미구현) | — | — | — | — |
| DOB-ALT-01 | Alertmanager Running | `vector(1)` Rule 추가 → 제거 | `/api/v2/alerts` | firing → resolved | 테스트 Rule Run |
| DOB-ALT-02 | Credential Secret, DNS | 위 Alert | 메일 수신·헤더 | firing / resolved 수신, SPF/DKIM/DMARC pass | GitOps PR #97/#101 |
| DOB-ALT-03 | (Deferred) | — | — | — | — |
| DOB-DSH-01 | sidecar label | Sync | `/api/search`, `/api/datasources` | 4종 로드, isDefault 1개 | GitOps PR #73/#77/#114 |
| DOB-DSH-02 | 실제 계측 Metric | Query 대조 | 패널별 결과 | 존재 Metric만 사용, 관점 분리 | GitOps PR #114 |
| DOB-AUT-01 | Role Merge | 2회 실행 | changed | 2회차 changed=0 | Ansible 출력 |
| DOB-FLT-01~04 | (미실행) | 06 F-10~F-13 정의 | — | — | — |
| DOB-Q07-01 | (미실행) | 수동 vs 자동 | 시간·수동 단계 | — | — |

---

## 10. Test 상세 판정

### DOB-REG-01 — Harbor Registry·TLS·Robot 분리

Preconditions: `harbor` Role 적용, 내부 CA 배포.

PASS:

- VM 재부팅 후 Harbor 자동 기동(systemd).
- BuildKit Push / Node Pull 모두 x509 오류 없음.
- CI Robot(Push/Pull), API Robot, Runtime pull-only Robot이 분리되어 있고 Pipeline에서 Admin Credential 미사용.

현재 결과: `PASS`. Evidence: TS-010 / TS-017 / TS-024, Infra PR #47 / #161 / #183.

### DOB-REG-02 — Tag Immutability + `scan-*` 예외

Stimulus: 동일 `git-*` Tag에 다른 Digest 재Push, `git-*` 삭제 시도, `scan-*` 생성·삭제.

PASS:

- 동일 Tag 재Push 거부.
- `git-*` 삭제 `412 PRECONDITION`.
- `scan-*` 생성 `201`, 삭제 `200`.

현재 결과: `PASS`. 1차 시도(`matches`+`excludes` 동시 사용)는 저장 `200`에도 예외가 무력화된 **실패 Run으로 보존**한다. Evidence: TS-027 / TS-034, Infra PR #110 / #157.

### DOB-CI-01 — PR Pipeline

PASS:

- App CI Check(Backend·OpenAPI·Frontend·Browser) PASS.
- Build Verify(Backend·Frontend) PASS.
- Harbor Push, GitOps 변경, Provider Credential 사용 없음.

현재 결과: `PASS`. Evidence: App PR #54 / #65, 이후 모든 App PR의 Jenkins PR Build.

### DOB-CI-02 — main Image Pipeline

Stimulus: `seokpan-app` main Merge.

Observation: Branch Guard, P1, Final Guard, Candidate Build/Push(SBOM/Provenance), Trivy, Health Smoke, Promote, Digest Verify, Candidate Cleanup, `image-metadata.json`.

PASS:

- 모든 Stage PASS.
- Trivy에서 CRITICAL / fixable HIGH 0.
- Health Smoke `/health/live` 200(backend·frontend).
- Final Digest == Candidate push-time Digest.
- `scan-*` Tag-only Cleanup 성공.
- `image-metadata.json`에 Commit / Build / Tag / Digest / Scan / Smoke 기록.

FAIL 조건 검증(Fail-closed): non-main 실행 거부, P1 실패 시 Image Stage 미실행.

현재 결과: `PASS`. Evidence: App PR #68, main Build #18(backend/frontend Scan PASS, Health Smoke PASS, Promote 200, Evidence 기록), 이후 Source Freeze Commit(`b774a40`)까지 동일 Pipeline으로 생성된 Digest가 Runtime에 배포됨.

### DOB-CI-03 — Jenkins 재현성

PASS:

- Controller / Agent Image가 Digest로 고정.
- `plugins-lock.txt` 76개로 Controller 재생성 시 동일 Plugin 설치.
- 재생성 후 Credential ID(`harbor-robot-account`, `harbor-api-robot-account`, `gitops-promotion-github`) 자동 등록.

현재 결과: `PASS`. 1차 배포(`plugins.lock` 확장자)로 Controller가 다운된 **실패 Run(GitOps PR #62)** 을 보존하며, 수정 Run(PR #65)과 Credential 재생성 Run(PR #127, Controller Pod 재생성 후 `gitops-promotion-github` 등록 확인)을 PASS 근거로 사용한다.

### DOB-CD-01 — Digest Pinning 배포

Cross-role: 정태훈 `KAI-DEL-01`과 같은 Run을 공유한다. 본 역할은 Harbor → Pull Credential → Argo CD 구간을 소유한다.

PASS:

- 실제 존재하는 `git-<hash>` Tag Pull 성공(Backend·Frontend).
- GitOps `kustomization.yaml` Digest 반영 → Argo CD Synced → Pod imageID 일치 → Ready.
- `git-pending` / `latest` 실제 Runtime 미사용.

현재 결과: `PASS`. Evidence: seokpan-gitops#57 Step 2~6, GitOps PR #93.

### DOB-CD-02 — Self-Heal

Stimulus: `kubectl scale deploy/frontend --replicas=3`(영구 데이터 영향 없음).

PASS:

- Argo CD OutOfSync 감지.
- Git Desired State(2)로 자동 복귀.
- deployment-controller scale 이벤트와 Argo sync 이벤트 시각 일치.

부가 관찰: Git에 선언되지 않은 annotation만 추가한 Drift는 Self-Heal 대상이 아님을 확인(정상 동작).

현재 결과: `PASS`, 복귀 수 초. Evidence: seokpan-gitops#57 Step 7.

### DOB-CD-03 — Git Revert Rollback

Preconditions: `apps-frontend` Synced/Healthy, Baseline Digest `sha256:123203a4…0b0701`.

Stimulus: 존재하지 않는 Digest를 `apps/frontend/kustomization.yaml`에 Merge(GitOps PR #119) → `git revert` PR Merge(GitOps PR #120).

Observation: 새 Pod `ImagePullBackOff`(kubelet `not found`), 가용 Replica, Revert 후 Argo revision·Health·Pod Digest.

PASS:

- 불량 Revision이 Runtime에 도달해 실패가 관찰됨.
- Revert Merge 후 Synced/Healthy, Deployment 2/2.
- Pod Digest가 Baseline과 일치.
- `kubectl` 직접 조작 없음.

현재 결과: `PASS`. 불량 Merge 10:04 → Revert Merge 10:15.

**함께 기록하는 관찰(FAIL 아님)**: 기본 RollingUpdate가 새 ReplicaSet 검증 전에 기존 ReplicaSet을 줄여 가용 Replica가 **2/2 → 1/2**로 떨어졌다. Rollback 경로 자체의 결함이 아니라 Deployment 전략 개선 후보(`maxUnavailable: 0`)로 정태훈에게 인계한다.

> **Cross-role 반영 요청**: 정태훈 12 문서의 `KAI-CD-02 — Git Revert Rollback`은 현재 `Not Tested`로 기록되어 있다. 본 Run(seokpan-gitops#57 Step 8, GitOps PR #119/#120)이 KAI-CD-02의 PASS 조건(검증된 Revision 복귀·Argo Sync·Runtime Ready·장기 Cluster patch 없음)을 충족하므로 해당 문서 현행화를 요청한다. Recovery time은 위 Merge 시각 기준값이며 Argo Synced 시각 기준 재측정은 필요 시 새 Run으로 수행한다.

### DOB-CD-04 — GitOps Promotion PR 자동화

Preconditions: `gitops-promotion-github` Credential, Deployment `seokpan.io/app-source-commit` annotation이 실제 배포 상태와 일치(부트스트랩 값 2건을 Harbor API + kubectl로 대조해 정정한 뒤 시작).

Stimulus: App main Merge → main Pipeline → Create GitOps Promotion PR Stage.

Observation: `test_promote_gitops.py` 결과, 결과 코드, PR 변경 파일, Digest·annotation 값, Co-author.

PASS:

- Production Agent에서 Unit Test 9/9 통과.
- 이미지 입력 경로 변경이 없는 Commit → `PROMOTION_NO_CHANGE`.
- 변경 Component만 PR에 포함.
- PR Digest == `image-metadata.json` Final Digest.
- auto-merge / main 직접 push 없음, Branch Protection Review 후 Merge.
- 중복 open PR 존재 시 `PROMOTION_OPEN_PR_REQUIRES_REVIEW`로 fail-closed(회귀 테스트로 검증).

현재 결과: `PASS`.

```text
Build #20 (App PR #103 merge 자체)  → PROMOTION_NO_CHANGE=1 (CI 파일만 변경, 정확 판정)
App b774a40 → GitOps PR #132 (backend + frontend)
App 4b803c3 → GitOps PR #133 (frontend only)
App b281976 → GitOps PR #134 (frontend only)
```

Evidence: seokpan-app#98 / PR #103, seokpan-gitops PR #127, #132~#134(`seokpan-jenkins` Co-author).

리뷰 과정의 판정 변경(보존): 초기 구현은 기존 open PR을 Branch 이름만으로 재사용했으나 "동일 promotion 대상인지 검증하지 않는 fail-closed 위반"으로 REQUEST_CHANGES를 받아, 항상 사람 확인을 요구하는 방식(B안)으로 교체했다.

### DOB-CD-05 — Argo CD Webhook

PASS:

- `webhook.github.secret` 적용 후 Route 공개(순서 준수).
- GitHub Recent Deliveries `200`.
- argocd-server가 push event를 수신해 Refresh.
- 기존 `game.seokpan.soldesk.store` 경로(`/`, `/api/v1`, `/ws/v1`) 회귀 없음.

현재 결과: `PASS`(동작). Polling 대비 **Merge → Synced 지연 단축 수치는 측정 Run이 없으므로 기록하지 않는다**. 필요 시 동일 변경을 Webhook 비활성/활성 두 조건으로 측정하는 새 Run을 정의한다.

Evidence: seokpan-gitops#117, Infra PR #209 / #210, GitOps PR #123.

### DOB-MET-01 — In-cluster Target

PASS:

- `activeTargets` > 0(Prometheus → kube-apiserver 6443 허용).
- Issue #177로 비활성화한 4종(controller-manager/scheduler/etcd/kube-proxy)을 제외한 Down Target 0.

현재 결과: `PASS`. 초기 Down 19건의 원인 3종(6443 Egress, 비표준 포트 Egress, config-reloader Ingress)과 비활성화 결정을 기록한다. Evidence: Issue #69 → GitOps PR #68 / #70, Issue #177 → GitOps PR #71.

### DOB-MET-02 — 외부 VM Exporter

PASS:

- `node-exporter-external` 7/7 UP.
- `mariadb-exporter` 2/2 UP.
- `harbor-metrics` 1/1 UP.
- `lb01-haproxy-exporter` UP(8404, 실IP 바인딩).
- VRouter 방화벽 규칙이 `vrouter_firewall` Role에 포함(재구축 재현성), 2회차 `changed=0`.

현재 결과: `PASS`.

판정 변경 이력(보존):

```text
#103 최초 전제 "EndpointSlice 부재로 Target 0/0"
→ 실측: EndpointSlice·Service·ServiceMonitor·NetworkPolicy 정상, Exporter 미설치가 원인

Issue #205 최초 추정 "VRouter 관리 인터페이스 ACL"
→ 이유빈 진단: 9100/tcp가 internal zone에만 있고 external zone 규칙 없음

Infra PR #206 1차안 "단독 fix Playbook"
→ REQUEST_CHANGES(재구축 시 재발) → vrouter_firewall Role 통합
```

Evidence: seokpan-gitops#103, Issue #205 / Infra PR #206, Infra PR #204 / #208, GitOps PR #106 / #111 / #112.

### DOB-MET-03 — Application Metric

Cross-role(정태훈 `KAI-OBS-01`). 본 역할은 ServiceMonitor 선택 label(`release: kube-prometheus-stack`)과 NetworkPolicy(8000) 경로를 제공했다.

현재 결과: `PASS`. Evidence: seokpan-gitops#91 / GitOps PR #107.

### DOB-LOG-01 — Pod Log

PASS:

- Alloy DaemonSet 5/5.
- `{namespace="application"}` 등 LogQL에서 실제 Stream 반환.
- label `namespace` / `pod` / `container` / `node_name` 일치.

현재 결과: `PASS`. Evidence: GitOps PR #81(`totalLinesProcessed` 증가 확인).

> Cross-role: 정태훈 `KAI-OBS-02`(Application Logs 최종 Consumer Evidence)는 본 수집 경로 위에서 Application Log 내용·민감정보 미노출·ingestion delay를 판정한다. 수집 경로 자체는 본 Test로 PASS다.

### DOB-LOG-02 — Node Journal

Preconditions: 격리 Canary Pod에서 SELinux AVC Denial 없음 확인.

PASS:

- 5 Node(cp-01/02/03, worker-01/02) 연속 수집.
- `node_name` / `unit` / `level` label 추출.
- 기존 `pod_logs` 경로 회귀 없음, Argo CD Synced/Healthy.

현재 결과: `PASS`. Evidence: GitOps PR #99.

### DOB-LOG-03 — 외부 VM Log

현재 `Not Implemented / Deferred`. 06 문서의 `alloy_linux` Role과 Loki 외부 수신 경로(NodePort 31000)가 존재하지 않는다. MVP Acceptance 필수 조건에 포함하지 않으며, 외부 VM 장애 분석은 Metric과 VM 로컬 Log에 의존한다.

### DOB-ALT-01 — Alert firing → resolved

Stimulus: `vector(1)` 임시 PrometheusRule 추가 → 제거.

PASS: `/api/v2/alerts`에서 firing 확인 → Rule 제거 후 resolved.

현재 결과: `PASS`.

### DOB-ALT-02 — E-mail 수신·인증

Preconditions: `alertmanager-smtp-credential`(Ansible), `alertmanager-config`(GitOps), Resend 도메인 인증(SPF/DKIM), Preflight PASS.

PASS:

- Alertmanager Pod에 `smtp-password` 마운트, `/api/v2/status`에 기대 설정 반영.
- firing / resolved E-mail 실제 수신(`seokpan@soldesk.store`).
- SPF / DKIM / DMARC pass.
- Watchdog 메일 미수신(null Route).

현재 결과: `PASS`. 첫 메일 스팸함 도착은 인증 실패가 아닌 신규 도메인 평판 문제로 판정. 수신자는 임시 개인 주소에서 팀 Zoho 주소로 교체된 뒤 재확인했다.

Evidence: Infra PR #187 / #189, GitOps PR #97 / #101 / #129.

### DOB-ALT-03 — 프로젝트 전용 Alert Rule

현재 `Deferred`. 복제 중단·지연 Rule은 PROJECT_CHANGES 2026-09-21에서 2차 이관이 확정됐고, M-축 KPI Rule은 임계값 근거(3.2절)가 없다. `critical-email` Route는 Rule 추가 시 사용할 수 있도록 준비되어 있다. 이 항목을 MVP Acceptance 필수 조건에 포함하지 않는다.

### DOB-DSH-01 — Provisioning

PASS:

- Dashboard 4종이 ConfigMap으로만 로드(UI 변경 아님).
- Datasource Prometheus / Alertmanager / Loki, `isDefault` 1개.
- 같은 ConfigMap을 두 파일이 소유하지 않음.

현재 결과: `PASS`. Evidence: GitOps PR #73 / #75 / #77 / #114.

### DOB-DSH-02 — 관점 분리·내용 정합

PASS:

- Cluster(Infra Overview) / 외부 인프라(VM) / Data(MariaDB) / 서비스(KPI) 관점 분리 — 멘토링 권고 충족.
- 모든 패널 Query가 실제 존재하는 Metric만 사용.
- 외부 VM / MariaDB 패널에서 Down 인스턴스 자동 제외.
- 목표 임계값 미설정.

현재 결과: `PASS`. MariaDB Replication 패널은 작성 시점에 Metric 노출 원인 미확정으로 보류했으며, TS-043 수정 후 조회가 가능해졌으므로 2차 추가 대상이다.

### DOB-AUT-01 — 본 영역 Ansible Role 멱등성

| Role / Playbook | 2회차 changed=0 확인 | 비고 |
| --- | --- | --- |
| `vrouter_firewall`(9100 rich rule) | 확인 | Issue #205 |
| `argocd_bootstrap`(server.insecure) | 확인 | Infra PR #209 |
| `alertmanager_secret` | 확인 | Infra PR #187 |
| `argocd_webhook_secret` | 확인 필요 | 필드 merge 방식 |
| `harbor` / `harbor_immutability` | 확인 필요 | Immutability 수렴 로직(PR #157) |
| `jenkins_secrets` | 확인 필요 | RISK Playbook |
| `alertmanager_smtp_preflight` | 해당 없음 | 매 실행 Job 생성·정리(설계상 changed) |

현재 결과: `Partial`. 전체 Role에 대한 동일 Revision 2회 실행 Evidence를 한 Run으로 묶는 작업이 남아 있다.

### DOB-FLT-01~04 — 장애 주입(F-10~F-13)

현재 `Not Tested`.

| Test | 06 정의 | 비고 |
| --- | --- | --- |
| DOB-FLT-01 | Harbor 중단: 실행 Pod 유지 vs 신규 Pull/Scale/Deploy 실패 분리 | — |
| DOB-FLT-02 | Jenkins 중단: Runtime 무영향·CI 재개·PVC 재연결 | TS-035 우발 장애 1회(약 9분)는 계획 시험이 아니므로 PASS 근거로 쓰지 않음 |
| DOB-FLT-03 | Alertmanager 중단: 게임 유지·통보 공백·복구 후 resolved | — |
| DOB-FLT-04 | Prometheus/Loki Node 유실: 관측 공백·Local 데이터 유실·Evidence 보존 | Local PV 설계상 유실 수용 |

실행하지 않은 결과를 PASS로 기록하지 않는다. 2차 프로젝트에서 수행 여부를 Go/No-Go로 결정한다.

### DOB-Q07-01 — CI/CD 수동 대비 시간

현재 `Not Tested`. 7.1절의 Merge 시각 기반 값만 존재한다. 정식 측정은 다음 두 구간을 분리한다.

```text
자동 구간: App Merge → Jenkins 완료 → Promotion PR created_at
사람 구간: PR created_at → merged_at (Review)
CD 구간  : merged_at → Argo Synced → Pod Ready
수동 비교: 동일 변경을 수동 Build/Push/Manifest 수정으로 수행한 시간·단계 수
```

---

## 11. 검증 Phase

### Phase 0 — Registry / CA

현재 결과: `Completed`.

```text
DOB-REG-01 PASS
→ DOB-REG-02 PASS
```

### Phase 1 — CI

현재 결과: `Completed`.

```text
DOB-CI-01 PASS
→ DOB-CI-02 PASS
→ DOB-CI-03 PASS
```

### Phase 2 — CD

현재 결과: `Completed`.

```text
DOB-CD-01 PASS
→ DOB-CD-02 PASS
→ DOB-CD-03 PASS
```

### Phase 3 — Delivery Automation (G-06 E2E)

현재 결과: `Completed`.

```text
DOB-CD-04 PASS (실제 PR 3건)
DOB-CD-05 PASS (지연 수치 Not Measured)
```

### Phase 4 — Observability (G-07)

현재 상태:

```text
DOB-MET-01 / 02 / 03: PASS
DOB-LOG-01 / 02: PASS
DOB-LOG-03: Not Implemented / Deferred
DOB-ALT-01 / 02: PASS
DOB-ALT-03: Deferred
DOB-DSH-01 / 02: PASS
```

### Phase 5 — 장애·비교·자동화 품질

현재 상태:

```text
DOB-AUT-01: Partial
DOB-FLT-01~04: Not Tested
DOB-Q07-01: Not Tested
```

### Absolute Gates

현재 유효한 절대 조건:

```text
F-10~F-13 미실행
→ DOB-FLT-01~04 PASS 금지, "Harbor/Jenkins/Alertmanager/Prometheus 장애에 강하다" 표현 금지

외부 VM Log 미구현
→ "전체 인프라 Log 수집" 표현 금지 (In-cluster 한정으로 표기)

전용 Alert Rule 미작성
→ "MariaDB 복제 장애 자동 통보" 표현 금지

Webhook 지연 미측정
→ "Sync 지연 N초 단축" 표현 금지

Q-07 미측정
→ "배포 시간 N% 단축" 표현 금지
```

---

## 12. Before → Change/Fault → After

### Delivery 자동화

```text
Before  (2026-09-16 이전)
  Harbor Digest 확정 후 사람이 GitOps Manifest 수정 → PR → Merge
  deployment.yaml image는 git-pending placeholder
Change
  Jenkins Create GitOps Promotion PR Stage + seokpan.io/app-source-commit annotation
After
  App Merge 후 영향 Component만 자동 PR (#132~#134), 사람은 Review/Merge만 수행
```

비교: 수동 편집 단계 수, 잘못된 Component 동시 변경 여부, Digest 불일치 여부.

### Rollback

```text
정상 Baseline (2/2, sha256:123203a4…)
→ 불량 Digest Merge (#119) → ImagePullBackOff, 가용 1/2
→ git revert Merge (#120) → 2/2, Baseline Digest
```

비교: 가용 Replica 최저값, 복구 시간, 수동 개입 단계.

### Prometheus Target

```text
Before: activeTargets 0 → 이후 19건 Down
Change: NetworkPolicy 6443 / 비표준 포트 / config-reloader, 4종 Scrape 비활성
After : 비활성 4종 외 Down 0, 외부 VM 7/7
```

### Alert

```text
Before: alertmanager-config에 SMTP placeholder, 수신 불가
Change: Resend + 도메인 인증 + Credential Secret 분리 + Watchdog null
After : firing/resolved 수신, SPF/DKIM/DMARC pass
```

---

## 13. 중단 기준

즉시 중단:

- Secret / Token / API Key 평문 노출
- Scan FAIL 상태에서 Promote 강행
- 다른 사람의 open Promotion PR이 있는데 새 Promotion을 강제로 생성하려는 시도
- selfHeal 대상 리소스를 `kubectl`로 수정해 결과로 인정하려는 시도
- 소유자 확인 없이 VRouter / LB / DB 서버 설정 변경
- Rollback 시험 중 영구 데이터(DB Schema 등)에 영향을 주는 변경 포함
- 장애 주입 시험이 게임 Runtime 가용성에 예상 밖 영향을 줄 때

중단 후:

```text
Run = FAILED / BLOCKED
→ 원인 기록
→ 영향 범위 고정
→ 11 Rollback / Recovery
→ 새 Run ID 재실행
```

---

## 14. 결과 기록 Template

```text
Test Case:
Run ID:
Target:
Date/Time (KST):
Operator:
State: PASS / FAIL / BLOCKED / NOT TESTED

App Commit:
GitOps Commit / Argo Revision:
Infra Commit:
Jenkins Build:

Preconditions:
Stimulus/Fault:
Observation:
Measured:
PASS/FAIL Basis:

Evidence:
- Issue/PR:
- Runbook/Command:
- image-metadata.json / promotion result:
- Prometheus / Loki Query:
- Alertmanager API / Mail Header:
- Timeline:

Failure/Exception:
Manual Steps:
Cleanup/Rollback:
Follow-up:
```

---

## 15. MVP Acceptance

Gate G(최종) 판정은 본 역할만으로 완료되지 않으며, Kubernetes/Application Integration, Data/Storage/Recovery, Network/External Infra와 Cross-role로 연결된다. 본 역할은 그 중 **M4 Delivery/Observability**와 **06 G-06·G-07**을 책임진다.

### 15.1 현재 본 역할 검증 상태

| 항목 | 현재 상태 |
| --- | --- |
| Harbor Registry / TLS / Robot / Immutability | PASS |
| PR Pipeline / main Image Pipeline | PASS |
| Jenkins 재현성 | PASS |
| Digest Pinning 배포(Cross-role) | PASS |
| Self-Heal(Cross-role) | PASS |
| Git Revert Rollback | PASS(가용성 1/2 저하 관찰) |
| GitOps Promotion PR 자동화(G-06 E2E) | PASS |
| Argo CD Webhook | PASS(동작), 지연 Not Measured |
| In-cluster / 외부 VM / Application Metric | PASS |
| Pod Log / Node Journal | PASS |
| 외부 VM Log | Not Implemented / Deferred |
| Alert firing → resolved / E-mail | PASS |
| 프로젝트 전용 Alert Rule | Deferred |
| Dashboard 4종 | PASS |
| Ansible 멱등성 Evidence 통합 | Partial |
| F-10~F-13 | Not Tested |
| Q-07 | Not Tested |

```text
06 G-06 Delivery       (Commit → Digest → Healthy)          → PASS
06 G-07 Observability  (Metric·Log 조회, Alert E-mail)       → PASS
07 M4 Delivery/Observability                                → PASS
```

### 15.2 2차 프로젝트로 넘기는 검증

- F-10~F-13 장애 주입 시험 Go/No-Go 및 실행
- 프로젝트 전용 Alert Rule(MariaDB 복제 중단·지연, Backend 5xx 등) — 임계값은 실측 Baseline 이후 정의
- 외부 VM Log 수집 경로(`alloy_linux` + Loki 외부 수신) 또는 미도입 근거 기록
- Webhook 유/무 Sync 지연, Q-07 CI/CD 구간 분리 측정
- Harbor Retention, Jenkins Job Git 선언, 레거시 Credential Secret 정리
- DOB-AUT-01 전체 Role 멱등성 Evidence 단일 Run 정리
- 정태훈 KAI-CD-02 현행화(본 문서 DOB-CD-03 Evidence 연결)
- `PROJECT_CHANGES.md`에 Jenkins → GitOps PR 자동화 확장(2026-09-18) 항목 추가

이미 PASS한 항목을 다시 미완료로 표현하지 않으며, 아직 실행하지 않은 장애 시험·전용 Alert·외부 VM Log 항목을 완료된 결과처럼 기록하지 않는다.

## 16. 문서 완료 기준

- 02 C-06 / 06 G-06·G-07·F-10~F-13·Q-07 Traceability 유지.
- 09 Gate A~G Traceability와 정합.
- 11 실행절차 반복 없이 Test Contract로 연결.
- 각 Test의 Preconditions / Stimulus·Fault / Observation / PASS·FAIL / Evidence 명확.
- 정량 항목의 단위와 측정 출처 명확, 관찰값과 목표값 구분.
- 미실행·미구현·보류 항목을 결과처럼 작성하지 않음.
- 판정이 바뀐 Test(REG-02, CI-03, CD-04, MET-02)의 변경 과정을 보존.
- Cross-role Owner 경계 명확(KAI-DEL-01, KAI-CD-01·02, KAI-OBS-01·02).
- Secret / Token / API Key Evidence 금지.
- 실패 Run 보존, 재실행은 새 Run ID.
