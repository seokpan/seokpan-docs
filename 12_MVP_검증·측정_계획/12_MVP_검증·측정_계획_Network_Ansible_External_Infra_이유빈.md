# MVP 검증·측정 계획

## Network / Ansible / External Infra

## 1. 목적과 문서 경계

이 문서는 「石나가는 판단」 1차 프로젝트의 Network / Ansible / External Infra 영역에서 **무엇을 시험하고, 무엇을 측정하며, 어떤 근거로 PASS/FAIL을 판정할지** 정의한다.

```text
09 = What / Why / Responsibility / Gate
11 = Pre-check / Apply / Verify / Re-run / Recovery / Rollback
12 = Test Case / Preconditions / Stimulus·Fault / Measurement / PASS·FAIL / Evidence
```

12는 11의 실행 명령을 그대로 반복하지 않는다. 실제 적용·복구 절차는 11 Runbook을 참조하고, 본 문서는 각 Test의 선행조건, 실행 자극 또는 장애, 관찰 항목, 측정값, PASS/FAIL 기준과 Evidence 규격을 정의한다.

직접 기준은 다음과 같다.

- [`02_SeokPan_핵심_문제_및_검증_목표.md`](../01-08_기획·설계_Baseline/02_SeokPan_핵심_문제_및_검증_목표.md)
- [`06_SeokPan_Ansible_자동화_테스트_설계.md`](../01-08_기획·설계_Baseline/06_SeokPan_Ansible_자동화_테스트_설계.md)
- [`07_SeokPan_확장_호환형_MVP_도출.md`](../01-08_기획·설계_Baseline/07_SeokPan_확장_호환형_MVP_도출.md)
- [`09_MVP_실행·통합_실시설계_Network_Ansible_External_Infra_이유빈.md`](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Network_Ansible_External_Infra_이유빈.md)
- [`10_GitHub_협업_및_Repository_운영.md`](../10_GitHub_협업_및_Repository_운영/10_GitHub_협업_및_Repository_운영.md)
- [`11_MVP_구축·자동화_Runbook_Network_Ansible_External_Infra_이유빈.md`](../11_MVP_구축·자동화_Runbook/11_MVP_구축·자동화_Runbook_Network_Ansible_External_Infra_이유빈.md)
- [`PROJECT_CHANGES.md`](../PROJECT_CHANGES.md)
- [`MVP_IMPLEMENTATION_BASELINE.md`](../MVP_IMPLEMENTATION_BASELINE.md)
- `seokpan-infra`의 실행 대상 Inventory·Role·Playbook·Safe Runner 및 관련 Issue·PR·Runtime Evidence
- Kubernetes / Application Integration, Database / Storage / Recovery, Delivery / Observability 영역의 공식 검증 결과

실제 Test Run 직전에는 실행하려는 `seokpan-infra` Revision, Inventory, Safe Runner, 열린 Issue·PR과 각 Consumer 영역의 최신 상태를 다시 확인한다.

과거 검증 기록은 해당 시점·대상·Revision의 Evidence로 재사용할 수 있으나 현재 Revision의 실행 성공으로 자동 승격하지 않는다.

---

## 2. 상위 검증축과 Gate 대응

상위 문서의 M-01~M-05 검증축을 대체하지 않는다.

Network / Ansible / External Infra 영역은 M-05 Ansible 개선을 핵심으로 소유하고, M-01~M-04에는 Network·Endpoint·Metric·공통 실행 기반을 제공하는 Cross-role로 참여한다.

| ID | 검증축 | 본 역할과의 관계 |
| --- | --- | --- |
| M-01 | 동시성 정확성 | Cross-role — Application Replica·DB Consumer가 통신할 수 있는 Network 경로 제공 |
| M-02 | Vote 처리 성능 | Cross-role — Network/LB 경로와 Metric 수집 경계 제공 |
| M-03 | Backend 장애 복구 | Cross-role — 재연결 시 공통 VIP·Route·Gateway 경로의 정상 여부 확인 |
| M-04 | DR | Cross-role — DB·Kubernetes Recovery 이후 공통 Network Endpoint 접근 경로 확인 |
| M-05 | Ansible 개선 | **핵심** — Project Runtime, Role 재실행, 멱등성, Drift 복원, 실패·차단 경계 측정 |

09 Integration Gate와 Test Case의 대응은 다음과 같다.

```text
Gate A — Project Runtime
  → NAE-RUN-01 / NAE-RUN-02

Gate B — VRouter Route
  → NAE-VR-01 / NAE-VR-02 / NAE-VR-03

Gate C — Firewall / NAT
  → NAE-FW-01 / NAE-FW-02

Gate D — Common Endpoint / LB
  → NAE-LB-01 / NAE-LB-02 / NAE-END-01 / NAE-END-02

Gate E — External Metric Network
  → NAE-MET-01 / NAE-MET-02

Gate F — Common Time
  → NAE-TIME-01

Gate G — Safe Ansible Execution
  → NAE-ARS-01 / NAE-ARS-02
```

추가 Cross-role Evidence는 다음과 같이 연결한다.

```text
Browser / HTTPS / WebSocket
→ Kubernetes / Application Integration 결과 재사용

DB VIP / Backend → MaxScale TLS
→ Database / Storage / Recovery 결과 재사용

Prometheus Target / HAProxy Stats
→ Delivery / Observability 결과 재사용

SSH Bootstrap #27
→ 현재 Planned, 별도 후속 검증
```

---

## 3. 상태와 측정 원칙

### 3.1 상태

본 문서에서는 다음 상태를 사용한다.

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
Not Implemented
Not Tested
Failed
```

다음을 같은 의미로 취급하지 않는다.

```text
Role 존재
≠ Check Mode 통과
≠ Actual Run 성공
≠ 정상 2회차 changed=0
≠ Drift 복원 성공

Route 존재
≠ Persistent Route 존재
≠ 왕복 통신 성공
≠ Application 기능 성공

HAProxy Listener
≠ Backend Reachable
≠ Consumer 기능 정상

Exporter 로컬 응답
≠ Network 접근
≠ Prometheus Target UP

--inspect-only 성공
≠ 대상 SSH 성공
≠ Become 성공
≠ Check Mode 성공
≠ Actual Apply 성공
```

### 3.2 임의 목표값 금지

다음 값은 실제 측정 없이 목표값으로 확정하지 않는다.

- 전체 Infra 구축 시간
- 수동 대비 Ansible 개선률
- Role별 허용 실행 시간
- 장애 복구 목표시간
- Network 전환 목표시간
- Safe Runner 추가 Overhead
- 허용 Packet Loss
- Endpoint 응답시간 Threshold

이미 과거 Run에서 측정된 값이 있더라도 해당 Revision·환경의 관찰값이다.

조건이 바뀌면 새 Run으로 측정하고 이전 값을 덮어쓰지 않는다.

### 3.3 현재 MVP 범위

현재 1차 MVP 범위는 다음과 같다.

- VRouter 4대
- 네 사설망
- `lb-01` 단일 LB
- Common VIP `10.1.93.90`
- HAProxy L4 전달
- Project Endpoint 게시
- Ansible Project 고정 실행환경
- Safe Runner 기반 승인 실행
- VRouter·LB Metric Network
- Chrony 공통 시간

다음은 현재 MVP PASS 조건으로 승격하지 않는다.

- `lb-02`
- Keepalived 자동 VIP Failover
- 신규 VM부터 전체 Clean Build
- Kubernetes API FQDN SAN·게시 완료
- SSH Bootstrap Issue #27 완료
- Safe Runner에서 현재 미검증 Playbook의 성공 가정
- P4 전체 Acceptance

---

## 4. Evidence 규격

각 Test Run은 최소 다음 Metadata를 남긴다.

```text
Test Case ID
Run ID
Target
Start Timestamp
End Timestamp
Operator

Docs Commit SHA
Infra Commit SHA
Infra Branch
Inventory Revision

Controller User
Python Version
ansible-core Version

Related Issue / PR
11 Runbook Section

Precondition State
Stimulus / Fault
Result State
Failure / Block Reason

Measured
Manual Steps
Cleanup / Rollback
Follow-up
```

필요하면 다음 Evidence를 함께 저장한다.

```text
ansible-inventory 결과
ars inspect 결과
Check Mode 결과
Actual Run recap
Second Run recap
Route Snapshot
NetworkManager Persistent Route
firewalld Runtime/Permanent
ip_forward
VIP Address/Prefix
HAProxy Listener
HAProxy Backend 상태
Endpoint Lookup
HTTP/TCP Consumer 결과
Prometheus Target
HAProxy Stats
Chrony tracking
timedatectl
Failure Log
Recovery Timeline
Screenshot
Checksum
```

Evidence에는 다음을 기록하지 않는다.

```text
Vault Password
Become Password
Token
Private Key
Kubeconfig Credential
Secret 실제 값
전체 DB Credential URL
```

### 4.1 Run ID

형식:

```text
<TEST-CASE-ID>-YYYYMMDD-HHMM-<short-sha>
```

예:

```text
NAE-VR-01-20260928-2130-a1b2c3d
```

### 4.2 실패 Run 보존

실패한 Run을 수정해서 PASS 기록으로 덮어쓰지 않는다.

```text
FAILED / BLOCKED Run 보존
→ 원인 기록
→ 수정 또는 Recovery
→ 새 Run ID
→ 재실행
```

---

## 5. 정량 측정 Catalog

| 항목 | 단위 | 측정 방법 / 출처 | 용도 |
| --- | --- | --- | --- |
| Playbook elapsed | s | Run 시작~종료 Timestamp | M-05 |
| changed Task | count | Ansible recap | 멱등성 |
| failed Task | count | Ansible recap | 실행 품질 |
| unreachable Host | count | Ansible recap | 대상 접근 |
| Manual intervention | count | Run Log | 자동화 개선 |
| Target host | count | Inventory / recap | 범위 추적 |
| Route mismatch | count | Runtime/Persistent 비교 | VRouter |
| Route recovery | count | Drift 주입 후 복원된 항목 | Drift |
| Firewall mismatch | count | Runtime/Permanent 비교 | Firewall |
| HTTP status | code | Consumer request | Endpoint |
| TCP connect | success/fail | Consumer 연결 | LB/DB/API |
| Prometheus Target | UP/DOWN | Prometheus Target | Metric |
| Chrony sync | yes/no | timedatectl / chronyc | Time |
| Recovery time | s | 장애/Drift→정상화 Timeline | 복구 |
| Safe Runner blocked | count | Runner 결과 | 안전 통제 |
| Secret exposure | count | Log 검토 | 목표 0 |

### 5.1 M-05 측정 원칙

M-05를 단순히 “Ansible을 사용했다”는 사실로 PASS 처리하지 않는다.

대표 대상에 대해 다음을 비교할 수 있어야 한다.

```text
첫 Actual Run
정상 2회차
의도적 Drift 후 재실행
실패 또는 차단 Run
Recovery 또는 Rollback
```

수동 구축 시간과의 정확한 Before/After 비교 Evidence가 없다면 “자동화 개선률”은 `Not Tested`로 둔다.

---

## 6. Dependency Evidence 재사용

Cross-role Test는 다른 담당자의 공식 Evidence를 재사용할 수 있다.

재사용 시 반드시 다음을 기록한다.

```text
Source 영역
Source Test 또는 Issue/PR
Source Revision
Source 시점
본 Test에서 재사용하는 사실
본 Test가 추가로 확인하는 경계
```

### 6.1 재사용 가능한 대표 Evidence

| Source | 재사용 경계 |
| --- | --- |
| Kubernetes / Application Integration | 외부 HTTPS, Backend API, WebSocket, Browser Runtime |
| Database / Storage / Recovery | DB VIP/FQDN, Backend→MaxScale TLS |
| Delivery / Observability | VRouter Metric Target, HAProxy Metric Target |
| Infra Issue/PR | VRouter 반복 실행, Firewall 수정, HAProxy/LB, Chrony |
| PROJECT_CHANGES | Project Runtime·Safe Runner 정책 |

### 6.2 재사용 금지 방식

다음 방식으로 Evidence를 확대하지 않는다.

```text
과거 Route PASS
→ 현재 Safe Runner Apply PASS로 변환 금지

Prometheus Target UP
→ Exporter 전체 기능 PASS로 변환 금지

HAProxy Listener 존재
→ Application E2E PASS로 변환 금지

DB VIP TCP 성공
→ DB Query·TLS 전체 PASS로 변환 금지

다른 담당자 PASS
→ Network 담당 단독 성과로 표현 금지
```

---

## 7. 현재 검증 기준 상태

현재 09·11과 기존 Runtime Evidence 기준 상태는 다음과 같다.

| 영역 | 현재 기준 | 12에서의 처리 |
| --- | --- | --- |
| Project Runtime | 기존 VM 범위 `Validated` | 사용자·Revision별 재검증 |
| VRouter Route | 과거 4대 반복 Actual Run `Validated` | 과거 Evidence 재사용 + 현재 Runner 별도 |
| Firewall / NAT | `Partial` | 현재 Source 및 Apply 결과 필요 |
| lb-01 Network / VIP | `Partial` | 현재 Check/Apply 결과 분리 |
| HAProxy | `Partial` | Listener·Backend·Consumer 분리 |
| Endpoint | 서비스·DB 일부 `Validated`, API FQDN 보류 | Endpoint별 개별 판정 |
| VRouter Metric | 과거 Target 범위 `Validated` | Network 접근과 Prometheus Consumer 분리 |
| HAProxy Metric | 과거 Target `Validated` | LB Metric 별도 Test |
| Chrony | 과거 대상 범위 `Validated` | 현재 대상 재확인 |
| Safe Runner Policy | 정책 구현·일부 검증 | Playbook별 호환성은 별도 |
| SSH Bootstrap #27 | `Planned` | PASS 금지 |
| Clean Build | `Not Tested` | PASS 금지 |
| lb-02 / Keepalived | `Deferred` | Acceptance 필수조건 아님 |

### 7.1 현재 Safe Runner 경계

| Playbook | 현재 판정 |
| --- | --- |
| `vrouter_network` | 미등록 `check_mode:false` Evidence로 현재 일반 `ars` Apply 차단 대상 |
| `vrouter_firewall` | 자동 Check Mode 및 일반 Apply 통과 여부 미검증 |
| `lb_network` | `Detect LB NetworkManager connection`의 `check_mode:false` Evidence는 등록돼 있으나 Playbook 전체 Check Mode·Apply는 별도 검증 필요 |
| `lb_haproxy` | 조회 Task와 Assert의 Check Mode 정합성·Apply 미검증 |

이는 “기능 실패가 확인됐다”는 의미가 아니다.

Safe Runner가 현재 Source 또는 Evidence 경계 때문에 실행을 중단한 경우 Test 결과는 `BLOCKED`로 기록한다.

아직 실행하지 않은 경우는 `NOT TESTED`로 기록하며 둘을 같은 상태로 취급하지 않는다.

---

## 8. Test Traceability Matrix

| Test Case | 목적 | Gate | 11 | 현재 상태 |
| --- | --- | --- | --- | --- |
| NAE-RUN-01 | Project 고정 실행환경 재현 | A | 4~5 | Validated — 기존 검증 범위 |
| NAE-RUN-02 | Inventory·대상 접근 | A | 4 | Run별 판정 |
| NAE-VR-01 | VRouter NIC·Forwarding·Route | B | 6 | 과거 PASS / 현재 Runner 분리 |
| NAE-VR-02 | 정상 2회차 멱등성 | B | 6 / 14 | 과거 PASS / Revision별 재측정 |
| NAE-VR-03 | Route Drift 복원 | B | 6 / 13~15 | 과거 PASS / 현재 재실행 분리 |
| NAE-FW-01 | zone·Policy·선택적 NAT | C | 7 | Partial |
| NAE-FW-02 | Worker→VRouter `9100/tcp` Network Boundary | C | 7 / 11 | 과거 PASS — Cross-VRouter Evidence 범위 |
| NAE-LB-01 | VIP·Prefix·Return Route | D | 8 | Partial |
| NAE-LB-02 | HAProxy Listener·Backend | D | 9 | Partial |
| NAE-END-01 | Game/DB/Harbor 등 Project Endpoint | D | 10 | Endpoint별 판정 |
| NAE-END-02 | Kubernetes API FQDN | D | 10 | Not Tested / `pending_api_san` |
| NAE-MET-01 | VRouter 4개 Prometheus Consumer | E | 11 | 과거 PASS |
| NAE-MET-02 | HAProxy Stats Prometheus Consumer | E | 11 | 과거 PASS |
| NAE-TIME-01 | Chrony 동기화 | F | 12 | 과거 PASS |
| NAE-ARS-01 | Safe Runner 정책 차단 | G | 5 | Partial / 정책 검증 |
| NAE-ARS-02 | 허용 Playbook 실행 호환성 | G | 5~12 | Partial |
| NAE-AUT-01 | 재실행·Drift·Recovery 종합 | M-05 | 13~15 | Partial |
| NAE-SSH-01 | SSH Bootstrap 복구 | 후속 | 16 | Planned |
| NAE-CLEAN-01 | 신규 VM 전체 Clean Build | 후속 | 16 | Not Tested |

`PASS`는 해당 Test Case의 정의된 범위에 실제 Runtime Evidence가 있을 때만 사용한다.

과거 특정 Revision의 PASS와 현재 Revision에서의 Safe Runner 재실행 가능성은 별도 상태로 유지한다.

---

## 9. Test Contract Matrix

| Test | Preconditions | Stimulus / Fault | Observation / Measurement | PASS 핵심 | Evidence |
| --- | --- | --- | --- | --- | --- |
| NAE-RUN-01 | 일반 사용자, 개인 checkout | Project 환경 생성·검증 | Python·Ansible·Collection | 고정 버전 일치, Smoke 성공 | Infra #84, #94 / PR #96 |
| NAE-RUN-02 | Inventory Parse 가능 | 대상 접근 검사 | Host 수, unreachable | 예상 대상 일치, 접근 가능 | Inventory / recap |
| NAE-VR-01 | VRouter 4대 | Route Role 검증 | NIC, forwarding, Runtime/Persistent Route | 목표와 일치 | Infra PR #103 / #134 |
| NAE-VR-02 | NAE-VR-01 PASS | 동일 Revision 재실행 | changed/failed/unreachable | 불필요 변경 없음, failed=0, unreachable=0 | Infra PR #134 |
| NAE-VR-03 | 승인된 대상 | Persistent Route Drift | Drift 전/후, Runtime 통신 | 목표 상태 복원 및 재실행 멱등 | Infra PR #134 추가 검증 Evidence |
| NAE-FW-01 | Route 정상 | Firewall 검증 | zone/policy/NAT | 목표 Source/Port·NAT 제외 일치 | Role Source / Runtime Evidence |
| NAE-FW-02 | Worker/VRouter 준비 | Worker→VRouter `:9100` | TCP/HTTP | Cross-VRouter 경로에서 대상 `external` zone 유입 및 HTTP 접근 성공 | Infra #205 / PR #206 |
| NAE-LB-01 | lb-01 접근 | VIP/Route 검증 | VIP Prefix, 네 Return Route | Inventory 목표와 일치 | `lb_network` Source / Runtime Evidence |
| NAE-LB-02 | LB Network 정상 | Listener/Backend 검증 | 80/443/6443/3306 | 의도한 Listener와 Backend 경로 확인 | `lb_haproxy` Source / Consumer Evidence |
| NAE-END-01 | Consumer 준비 | Endpoint 접속 | DNS/hosts/TCP/HTTP | Endpoint별 정의된 Consumer 성공 | Infra #188 / Cross-role Evidence |
| NAE-END-02 | API SAN·게시 완료 | API FQDN 접속 | TLS/SAN/Client | 인증서·이름·접속 모두 성공 | Client Evidence |
| NAE-MET-01 | 4 VRouter Exporter 및 Network 경로 | Prometheus Scrape | VRouter 4개 Target | 정의된 4개 VRouter Target UP | Infra PR #206 / Delivery·Observability 12 |
| NAE-MET-02 | HAProxy Stats 활성 | Prometheus Scrape | `10.1.93.78:8404` | HAProxy Target UP | Infra PR #208 / Delivery·Observability 12 |
| NAE-TIME-01 | Chrony 설치 | 상태·동기화 확인 | active/enabled/sync/source/config | 모든 대상 기준 충족 | Infra #39 / PR #40 |
| NAE-ARS-01 | 일반 허용 사용자 | 금지 실행 시도 | 차단 여부 | 금지 경로 차단, 정상 정책 경로 유지 | Infra #160 / PR #184 |
| NAE-ARS-02 | Credential·Revision 준비 | 허용 Playbook 실행 시도 | Inspect/Target/Check/Approval/Apply | 정상 경로 전체 진행 시 PASS, 정책 차단 시 BLOCKED | Runner Evidence |
| NAE-AUT-01 | 대표 Role, 동일 Revision | 첫 Run→2회차→Drift→Recovery | 시간·changed·failed·manual step | 각 단계가 독립 Evidence로 설명 가능 | Run Series |
| NAE-SSH-01 | #27 구현 완료 후 | Key 누락/정상 혼합 | Matrix, 변경 대상 | 정상 SKIP, 누락만 변경, 2회차 멱등 | Infra #27 Evidence |
| NAE-CLEAN-01 | 신규 VM 준비 | 전체 Infra 구축 | 구축시간·실패·개입 | 별도 정의된 전체 구축 기준 충족 | Full Run |

---

## 10. Test 상세 판정

### 10.1 NAE-RUN-01 — Project Runtime

목적은 Controller마다 System Python이나 임의 Ansible이 아니라 Project 고정 실행환경을 재현할 수 있는지 확인하는 것이다.

Preconditions:

- 허용된 일반 사용자
- 개인 Infra checkout
- 실행 Revision 고정
- Secret 비노출

관찰:

```text
Python 3.12.13
ansible-core 2.20.8
kubernetes.core 6.5.0
ansible.mariadb 6.0.2
Python kubernetes 36.0.3
```

PASS:

- 필요한 Project Runtime이 동일 Revision에서 재현된다.
- Project Inventory Parse와 최소 Smoke가 성공한다.
- System Python·System Ansible을 Project 환경이 덮어쓰지 않는다.
- Secret이 로그나 argv에 노출되지 않는다.

현재 결과: `PASS — 기존 검증 범위`.

Evidence:

- `seokpan-infra#84` — 빈 Project 환경 Bootstrap 재현성, Inventory/Ping/Syntax/Smoke
- `seokpan-infra#94` / PR #96 — `ansible.mariadb 6.0.2` Collection Version Lock 및 재현성

과거 빈 환경 검증은 재사용할 수 있지만 다른 사용자·다른 checkout의 현재 성공으로 확대하지 않는다.

### 10.2 NAE-RUN-02 — Inventory / Target Access

목적은 실제 Test 대상과 Inventory가 일치하고 변경 실행 전 대상 접근이 가능한지 확인하는 것이다.

Preconditions:

- 실행 Revision 고정
- Inventory Parse 성공
- 예상 Test 대상 정의

Observation:

```text
Inventory Group
Target Host
ansible_host
Target Count
Target Connect
unreachable
```

PASS:

- 예상 Group과 Host가 Inventory에 존재한다.
- Test 대상 수가 계획과 일치한다.
- 예상하지 않은 Host가 `--limit` 또는 Play 대상에 포함되지 않는다.
- 필요한 대상 접근이 가능하다.

BLOCKED:

- Inventory 대상과 Test 계획이 다르다.
- 예상하지 않은 Host가 포함된다.
- SSH/Connection 실패로 대상 상태를 신뢰할 수 없다.

이 Test는 실제 실행마다 다시 판정한다.

### 10.3 NAE-VR-01 — VRouter Route

대상:

```text
vrouter-01  10.1.93.71 / 192.168.51.10
vrouter-02  10.1.93.73 / 192.168.52.10
vrouter-03  10.1.93.75 / 192.168.53.10
vrouter-04  10.1.93.77 / 192.168.54.10
```

NIC 자동 탐지는 다음 경계를 따른다.

```text
External NIC
= Inventory ansible_host와 동일한 IPv4를 가진 Interface

Internal NIC
= 192.168.51.10 ~ 192.168.54.10 중 해당 Host의 사설 Gateway 주소를 가진 Interface

External / Internal Candidate
= 각각 정확히 1개이며 서로 다른 Interface
```

판정은 세 층으로 분리한다.

```text
NIC·Forwarding
Runtime Route
Persistent Route
```

PASS는 세 상태가 실행 Revision의 Inventory 목표와 모두 일치할 때만 가능하다.

현재 결과: `PASS — 과거 Revision Evidence`.

Evidence:

- Infra PR #103 — NIC Fact 자료형 및 Check Mode 조회 보완
- Infra PR #134 — Persistent Route 멱등성, Runtime/Persistent Route 확인

현재 `vrouter_network`의 Safe Runner 호환성은 별도 `NAE-ARS-02`에서 판정한다.

과거 Actual Run PASS를 현재 일반 `ars` Apply PASS로 변환하지 않는다.

### 10.4 NAE-VR-02 — 정상 2회차

동일한 Revision, Inventory, 대상에서 첫 Actual Run 이후 재실행한다.

PASS 기준:

```text
failed=0
unreachable=0
불필요한 변경 없음
```

대표적인 strict idempotency 조건은:

```text
changed=0
failed=0
unreachable=0
```

이다.

다만 조회성 Task나 정책상 항상 변경되는 Task가 존재한다면 그 이유를 Source와 함께 기록하고 무조건적인 `changed=0` 주장을 하지 않는다.

현재 결과: `PASS — 과거 Revision Evidence`.

Evidence:

- Infra PR #134
- 당시 VRouter 4대 Actual Run 및 반복 실행에서 `changed=0 / failed=0 / unreachable=0`

### 10.5 NAE-VR-03 — Route Drift 복원

승인된 대상에서 복원 가능한 Drift만 사용한다.

대표 검증:

```text
정상 Persistent Route
→ Route 1건 의도적 누락
→ vrouter_network Actual Run
→ 누락 Route 복원
→ 정상 상태 재실행
```

PASS:

- Current != Desired 상태에서 변경 Task가 실제 실행된다.
- 누락 또는 불일치 Route가 Desired State로 복원된다.
- Runtime/Persistent Route 검증이 통과한다.
- 복구 후 재실행에서 불필요한 변경이 없다.
- 기존 내부망 통신에 회귀가 없다.

현재 결과: `PASS — 과거 Revision Evidence`.

Evidence:

- Infra PR #134 추가 검증
- `vrouter-01` Persistent Route 1건 누락
- Actual Run `changed=1`
- 누락 Route 복구
- 복구 후 재실행 `changed=0 / failed=0 / unreachable=0`
- 다른 내부망 Gateway 통신 정상

현재 Revision에서 동일 Test를 수행하면 새 Run ID를 사용한다.

### 10.6 NAE-FW-01 — Firewall / NAT

확인 대상:

- 외부 NIC → `external`
- 내부 NIC → `internal`
- `int-to-ext`
- `ext-to-int`
- Worker CIDR의 필요한 Source·Port
- 사설망 목적지 NAT 제외
- 그 외 목적지의 선택적 Masquerade

현재 Source의 핵심 기준:

```text
private_network = 192.168.0.0/16

rule family="ipv4"
destination not address="192.168.0.0/16"
masquerade
```

따라서 선택적 NAT를 단순히 “인터넷 전용 NAT”라고 표현하지 않는다.

PASS:

- NIC와 zone이 의도한 값이다.
- 필요한 Policy가 존재한다.
- 사설망 목적지에 원치 않는 Masquerade가 적용되지 않는다.
- 필요한 Source/Port 허용이 Runtime/Permanent 상태와 일치한다.

현재 결과: `Partial`.

현재 전체 Safe Runner Check/Apply Evidence가 없으므로 Gate C 전체를 PASS로 만들지 않는다.

### 10.7 NAE-FW-02 — Worker → VRouter `9100/tcp` Network Boundary

이 Test는 **Firewall/Network 경계까지만** 소유한다.

대상 Source:

~~~text
worker-01 = 192.168.51.30
worker-02 = 192.168.52.30

허용 CIDR:
192.168.51.0/24
192.168.52.0/24
~~~

대상 VRouter Metric 주소:

~~~text
192.168.51.10:9100
192.168.52.10:9100
192.168.53.10:9100
192.168.54.10:9100
~~~

Worker→VRouter 접근은 Source와 Target의 위치에 따라 대상 VRouter로 유입되는 NIC와 zone이 다르다.

~~~text
같은 사설망의 자기 VRouter 접근

worker-01(192.168.51.30)
→ vrouter-01(192.168.51.10)
→ 대상 VRouter의 internal NIC / internal zone

worker-02(192.168.52.30)
→ vrouter-02(192.168.52.10)
→ 대상 VRouter의 internal NIC / internal zone


다른 사설망의 VRouter 접근

예:
worker-01
→ 192.168.52.10 / 192.168.53.10 / 192.168.54.10
→ Source VRouter를 거쳐 라우팅
→ 대상 VRouter의 external NIC / external zone으로 유입
~~~

따라서 `external zone`의 Worker CIDR `9100/tcp` 허용 규칙은 모든 Worker→VRouter 접근을 위한 규칙이 아니다.

이 규칙은 **다른 사설망에서 대상 VRouter로 들어오는 Cross-VRouter 트래픽에 필요한 최소 허용 경계**로 해석한다.

향후 실제 Run에서는 Source→Target 조합별로 다음 정보를 함께 기록한다.

~~~text
Source IP
Target VRouter IP
ip route get 결과
Target ingress NIC
Target ingress zone
HTTP/TCP 결과
~~~

PASS:

- Source→Target 조합의 실제 경로와 대상 VRouter의 ingress NIC/zone을 설명할 수 있다.
- 같은 사설망의 자기 VRouter 접근은 대상 VRouter의 `internal` 경로로 확인된다.
- 다른 사설망의 VRouter 접근은 대상 VRouter의 `external` 경로로 확인된다.
- Cross-VRouter 접근에 필요한 Worker Source CIDR의 `9100/tcp` 최소 허용 규칙이 존재한다.
- 허용 범위를 불필요하게 전체 Source로 확대하지 않는다.
- Role 재실행 시 정상 상태를 불필요하게 변경하지 않는다.

현재 결과: `PASS — 과거 Cross-VRouter Runtime Evidence 범위`.

Evidence:

- Infra #205 — `worker-01`에서 `vrouter-02/03/04`의 `52.10/53.10/54.10:9100` 접근 실패 원인 조사
- Infra PR #206 — Cross-VRouter 트래픽이 대상 VRouter의 `external` zone으로 유입되는 경로에 Worker CIDR 최소 허용 통합
- 당시 `worker-01 → 52.10/53.10/54.10:9100` HTTP 200 확인
- Role 재실행 시 변경 없음 확인

위 Historical Evidence는 **Cross-VRouter 경로에 대한 검증 범위**다.

`worker-01 → vrouter-01` 또는 `worker-02 → vrouter-02`와 같은 같은 사설망의 자기 VRouter 접근을 `external zone` Evidence로 확대하지 않는다.

Prometheus에서 실제 Target이 UP인지의 최종 Consumer 판정은 `NAE-MET-01`에서 수행한다.

### 10.8 NAE-LB-01 — lb-01 Network / VIP

확인 대상:

```text
Management IP = 10.1.93.78
Common VIP    = 10.1.93.90/24
```

Return Route:

```text
192.168.51.0/24 → 10.1.93.71
192.168.52.0/24 → 10.1.93.73
192.168.53.0/24 → 10.1.93.75
192.168.54.0/24 → 10.1.93.77
```

PASS:

- VIP Address뿐 아니라 Prefix까지 일치한다.
- 네 Return Route의 목적지와 Next Hop이 모두 일치한다.
- Runtime Route와 Persistent Route의 차이를 설명할 수 있다.

현재 결과: `Partial`.

`lb_network`는 `+ipv4.routes`, `+ipv4.addresses` 방식의 additive 설정을 사용한다.

따라서 오래된 Route 또는 VIP 값이 Desired State에서 제거되었을 때 자동으로 정리되는지는 별도 Drift/Rollback Evidence 없이는 PASS로 간주하지 않는다.

### 10.9 NAE-LB-02 — HAProxy

Listener:

```text
80
443
6443
3306
```

Backend:

```text
80
→ worker-01/02 :30080

443
→ worker-01/02 :30443

6443
→ cp-01/02/03 :6443

3306
→ maxscale-01 :3306
```

Stats는 서비스 VIP와 별도다.

```text
10.1.93.78:8404
```

PASS는 Listener 존재만으로 판정하지 않는다.

```text
Listener
→ Backend Reachability
→ 실제 Consumer
```

를 분리한다.

현재 결과: `Partial`.

HAProxy Role에는 Check Mode에서 조회 `command`가 skip될 경우 후속 Assert가 신뢰할 수 없게 되는 구조적 제약이 확인돼 있으므로 현재 Playbook 전체의 Check Mode/Actual Apply 상태는 별도 `NAE-ARS-02`에서 다룬다.

### 10.10 NAE-END-01 — Project Endpoint

Endpoint는 Host·CoreDNS·외부 Client 게시 여부가 서로 다르므로 하나의 전체 PASS로 뭉치지 않는다.

현재 대표 상태:

| Endpoint | Address | 현재 경계 |
| --- | --- | --- |
| Harbor | `192.168.53.61:443` | Host/CoreDNS 게시 |
| DB | `10.1.93.90:3306` | Host/CoreDNS 게시 |
| Game | `10.1.93.90:80/443` | Host/CoreDNS/Client 게시 |
| Grafana | `10.1.93.90:443` | `runtime_pending` |
| Kubernetes API | `10.1.93.90:6443` | `pending_api_san` |
| Jenkins | `10.1.93.90:443` | `deferred_route` |
| Argo CD | `10.1.93.90:443` | `deferred_external_ui` |

Game External Endpoint의 현재 확인 범위는 Cross-role Evidence를 재사용한다.

Evidence:

- Infra #188 — `game.seokpan.soldesk.store`의 Production TLS와 Windows Host/Linux VM 경로
- Kubernetes / Application Integration의 HTTPS·Backend API·WebSocket Runtime Evidence

DB Consumer 결과는 Database / Storage / Recovery의 DB VIP/FQDN 및 Backend→MaxScale TLS Evidence를 재사용한다.

다른 담당자의 Consumer 성공을 Network 담당의 단독 성과로 표현하지 않는다.

### 10.11 NAE-END-02 — Kubernetes API FQDN

현재 상태는 `Not Tested / pending_api_san`.

실제 SAN 정합화와 이름 게시가 끝나기 전에는 PASS로 승격하지 않는다.

PASS 조건:

- API 인증서 SAN에 사용 FQDN 존재
- 의도한 Host/CoreDNS/Client 게시 완료
- Client TLS 검증 성공
- Kubernetes API 요청 성공

IP `10.1.93.90:6443` Listener가 존재하는 사실은 이 Test의 PASS가 아니다.

### 10.12 NAE-MET-01 — VRouter Metric Consumer

이 Test는 `NAE-FW-02`에서 확보한 Network 경로를 Delivery / Observability Consumer까지 연결한다.

대상은 다음 네 VRouter다.

```text
192.168.51.10:9100
192.168.52.10:9100
192.168.53.10:9100
192.168.54.10:9100
```

다음을 구분한다.

```text
Exporter local response
→ Worker에서 :9100 Network 접근
→ Prometheus VRouter Target UP
```

PASS:

- 네 VRouter의 Metric Endpoint가 정의된 주소로 Scrape 가능하다.
- Prometheus에서 네 VRouter Target이 기대 상태로 확인된다.
- Target 이름·주소가 Inventory와 일치한다.

현재 결과: `PASS — 과거 Target Evidence`.

Evidence:

- Infra #205 / PR #206 — Firewall 경로 복원
- Delivery / Observability 12 `DOB-MET-02`

과거 `node-exporter-external 7/7 UP`에는 다음 대상이 함께 포함된다.

```text
VRouter 4대
MaxScale
NFS
Ansible Controller
```

따라서 `7/7`을 “VRouter 7대”라고 해석하지 않는다.

### 10.13 NAE-MET-02 — HAProxy Metric Consumer

HAProxy Metric은 VRouter `node_exporter` Target과 별도다.

대상:

```text
10.1.93.78:8404
```

PASS:

- HAProxy Native Prometheus frontend가 `10.1.93.78:8404`에 바인딩된다.
- `/metrics`가 정상 응답한다.
- Cluster 내부 Consumer에서 접근 가능하다.
- Prometheus Target이 UP이다.
- 기존 80/443/6443/3306 Listener에 회귀가 없다.

현재 결과: `PASS — 과거 Runtime Evidence`.

Evidence:

- Infra PR #208
  - `10.1.93.78:8404/metrics` HTTP 200
  - Cluster 내부 접근 HTTP 200
  - `lb01-haproxy-exporter` Target UP
- Delivery / Observability 12 `DOB-MET-02`

현재 Role 기본값은 Stats 비활성이며 `lb-01`만 host_vars에서 명시적으로 활성화한다.

### 10.14 NAE-TIME-01 — Chrony

최소 확인:

```text
Chrony package 존재
chronyd active
chronyd enabled
NTPSynchronized=yes
Leap status Normal
선택된 NTP Source 존재
공통 NTP Source 설정 일치
```

현재 공통 설정 기준:

```text
pool 2.centos.pool.ntp.org iburst
```

현재 결과: `PASS — 과거 검증 대상`.

Evidence:

- Infra #39
- Infra PR #40
- 당시 Harbor 포함 전체 16대
- Check Mode `changed=0 / failed=0 / unreachable=0`
- Actual Run 1차 및 2차
- 2회차 멱등성 확인

현재 Run 대상 전체가 같은 조건을 만족할 때만 새 Run PASS로 기록한다.

### 10.15 NAE-ARS-01 — Safe Runner Policy

검증 대상:

- 허용 사용자
- root 실행 차단
- sudo 실행 차단
- setuid 실행 차단
- Python Version 불일치 차단
- Keyring 이상 차단
- Vault 필요 여부 판정
- Become 필요 여부 판정
- Secret 비노출
- 영향 큰 Playbook 재승인
- 미검토 `check_mode:false` Fail-Closed

PASS는 “차단이 많다”가 아니다.

```text
허용된 정상 경로
→ 정상 진행

금지 또는 미검토 경로
→ 의도대로 차단
```

이 둘이 함께 성립해야 정책 PASS로 본다.

현재 상태: `Partial`.

Evidence:

- Infra #160
- Infra PR #184
- 현재 `ansible-safe-run` Source

정책 구현 자체와 모든 사용자×모든 Playbook Apply 검증은 구분한다.

### 10.16 NAE-ARS-02 — Playbook Compatibility

Playbook마다 별도로 판정한다.

```text
vrouter_network
vrouter_firewall
lb_network
lb_haproxy
```

`--inspect-only`만 성공한 상태를 PASS로 기록하지 않는다.

Test 단계:

```text
Inspect
→ Target Connect
→ Check / Preflight
→ Approval
→ Actual Run
→ Second Run
```

판정:

```text
PASS
= 해당 Playbook이 현재 Revision에서
  정책에 맞게 Inspect부터 Actual Run까지 진행되고
  필요한 검증과 정상 2회차까지 완료됨

BLOCKED
= Safe Runner가 미검토 check_mode:false,
  Source/Evidence 불일치 또는 정책상 위험 때문에
  의도적으로 Actual Run 이전에 중단함

FAIL
= 정책상 허용돼야 하는 정상 경로가 Runner 오류로 진행되지 못하거나,
  금지 경로가 허용되거나,
  실제 Apply가 실패함

NOT TESTED
= 해당 Revision에서 실제 실행 시도 Evidence가 없음
```

현재 기준:

- `vrouter_network` — 현재 일반 `ars` 경로에서 미등록 `check_mode:false` Evidence 때문에 `BLOCKED` 경계
- `vrouter_firewall` — 현재 자동 Check Mode / 일반 Apply 미검증
- `lb_network` — `Detect LB NetworkManager connection` Evidence는 현재 Safe Runner에 등록, Playbook 전체 Apply는 별도 검증 필요
- `lb_haproxy` — Check Mode 조회/Assert 정합성과 Actual Apply 별도 검증 필요

Safe Runner가 차단한 Playbook을 직접 `ansible-playbook`으로 우회해서 PASS를 만들지 않는다.

### 10.17 NAE-AUT-01 — Ansible 재실행·Drift·Recovery 종합

M-05의 핵심 종합 Test다.

Preconditions:

- 동일 Infra Revision
- 동일 Inventory
- 동일 대상
- 실행 가능한 Safe Runner 상태
- Before Snapshot 확보

Test Series:

```text
Run 1
첫 Actual Run

Run 2
정상 상태 재실행

Run 3
승인된 Drift 발생

Run 4
Drift 복원

Run 5
필요 시 Recovery / Rollback
```

측정:

- elapsed
- changed
- failed
- unreachable
- manual intervention
- Drift 항목 수
- 복원 항목 수
- Recovery time

PASS:

- 각 단계가 독립된 Run ID로 기록된다.
- 정상 2회차에 불필요한 변경이 없다.
- 의도한 Drift가 Desired State로 복원된다.
- 다른 정상 상태를 불필요하게 변경하지 않는다.
- 실패 또는 차단 Run을 삭제하거나 PASS로 덮어쓰지 않는다.
- Recovery/Rollback 이후 재검증 결과가 남는다.

현재 상태: `Partial`.

VRouter Persistent Route의 정상 2회차·Drift 복원은 Infra PR #134에서 Evidence가 있다.

하지만 Network / Ansible / External Infra 전체 Role에 대한 동일 조건의 종합 M-05 Run은 완료되지 않았다.

수동 구축 대비 정량 개선률도 현재 `Not Tested`다.

### 10.18 NAE-SSH-01 — SSH Bootstrap 복구

현재 상태: `Planned`.

Evidence Source:

- Infra #27 — 현재 Open

향후 PASS 기준:

- 허용 사용자×대상 VM Matrix 기록
- 기존 공개키가 정상인 조합은 `SKIP`
- 누락된 사용자 공개키만 추가
- 중복 Key 추가 없음
- 공개키 누락이 아닌 SSH 오류는 자동 재배포하지 않음
- 배포 후 SSH 접속 재검증
- 전체 정상화 후 2회차 실행에서 정상 조합 `SKIP`
- Host Key 이상은 자동 무시하지 않음
- Secret 비노출

Issue #27이 Open 상태인 현재는 PASS로 기록하지 않는다.

### 10.19 NAE-CLEAN-01 — 신규 VM 전체 Clean Build

현재 상태: `Not Tested`.

이 Test는 빈 Project `.venv` 재현성이나 기존 VM 대상 Smoke Test와 다르다.

```text
빈 Project 실행환경 재현
≠ 신규 VM 전체 Infra Clean Build
```

향후 실행 시 다음을 측정한다.

- VM/SSH 선행조건
- 전체 구축 시작/종료 시각
- 각 Playbook 단계
- failed
- unreachable
- manual intervention
- Recovery
- 최종 Consumer 연결

현재 전체 Clean Build Runtime Evidence가 없으므로 PASS로 표현하지 않는다.

---

## 11. 검증 Phase

### Phase 0 — Freeze

다음을 고정한다.

- Docs Revision
- Infra Revision
- Inventory
- Operator
- 대상
- Consumer Revision

### Phase 1 — Read-only Precondition

변경 없이 현재 상태를 수집한다.

```text
Project Version
Inventory
Route
Firewall
VIP
Listener
Endpoint
Metric Target
Chrony
Safe Runner State
```

### Phase 2 — Isolated / Limited Test

가능한 경우 단일 VRouter 또는 제한 대상부터 검증한다.

Safe Runner의 정책 또는 Source가 제한 실행을 허용하지 않으면 임의 우회하지 않는다.

### Phase 3 — Actual Apply

승인된 Test만 Actual Run을 수행한다.

실행 전·후 Snapshot과 Recap을 보존한다.

### Phase 4 — Re-run

동일 Revision·대상에서 정상 2회차를 확인한다.

### Phase 5 — Drift / Fault

승인된 복원 가능한 Drift만 주입한다.

### Phase 6 — Consumer

Network 자체 상태 이후 실제 Consumer Evidence를 연결한다.

### Phase 7 — Recovery / Rollback

11 Runbook 기준으로 복구하고 새 Run ID로 재검증한다.

---

## 12. Before → Change/Fault → After

### VRouter Route

```text
Before
Runtime/Persistent Route 정상

→ Fault
승인된 Route 하나 누락 또는 차이

→ After
Role 재실행 후 목표 Runtime/Persistent Route 복원
```

비교:

- Route 개수
- 목적지
- Next Hop
- Persistent 설정
- SSH 유지
- Consumer 통신

대표 Historical Evidence는 Infra PR #134에 존재한다.

### Firewall

```text
Before
Worker→VRouter :9100 Network 접근 정상

→ Fault
검증 가능한 범위에서 Source/Port 정책 Drift

→ After
Firewall Network Boundary 복원
```

비교:

- Source CIDR
- Port
- Runtime/Permanent
- HTTP 접근

Prometheus Target은 별도 `NAE-MET-01`에서 판정한다.

운영 영향이 예상되면 실제 Fault 주입 대신 기존 Incident Evidence를 사용할 수 있다.

### LB Network

```text
Before
VIP/Return Route 정상

→ Fault
승인된 격리 조건의 Route/VIP Drift

→ After
목표 상태 복원
```

비교:

- VIP Address
- Prefix
- Route Destination
- Next Hop
- Runtime/Persistent

additive 구성의 stale 값 제거가 자동으로 보장되는지 별도로 기록한다.

### Safe Runner

```text
허용 사용자 정상 경로
→ 정상 Playbook
→ 사전 점검·승인·Actual Run

허용 사용자
→ 미검토 또는 금지 실행 조건
→ BLOCKED

root / sudo / setuid / 비허용 사용자
→ 차단
```

정상 경로와 차단 경로를 같은 PASS 기준으로 섞지 않는다.

---

## 13. 중단 기준

즉시 중단 조건:

- 예상하지 않은 대상이 Inventory에 포함됨
- 실행 Revision이 Test 기록과 다름
- NIC 자동 탐지 결과가 실제 Interface와 다름
- SSH 복구 접근 경로가 없음
- Default Route 또는 관리 IP가 예상 밖으로 바뀜
- Secret / Password / Token / Private Key 노출
- 미검토 `check_mode:false` Task가 Runner 정책에 의해 차단됨
- Check Mode의 조회값 누락으로 후속 Assert가 신뢰 불가능함
- VIP/Return Route 차이를 설명할 수 없음
- 다른 담당 영역의 Runtime을 임의로 변경해야만 Test가 진행됨
- 운영 서비스에 예상하지 않은 영향이 발생함

중단 후:

```text
Run = FAILED 또는 BLOCKED
→ 원인 기록
→ 영향 범위 고정
→ 11 Recovery / Rollback
→ Source 또는 Evidence 정합화
→ 새 Run ID
→ 재실행
```

다음은 금지한다.

```text
Safe Runner 차단
→ 직접 ansible-playbook으로 우회

실패 Run
→ 로그 삭제
→ 같은 Run ID를 PASS로 수정
```

---

## 14. 결과 기록 Template

```text
Test Case:
Run ID:
Target:
Date/Time (KST):
Operator:

State:
PASS / FAIL / BLOCKED / NOT TESTED

Docs Commit:
Infra Commit:
Infra Branch:
Inventory Revision:

Controller User:
Python:
ansible-core:

Preconditions:

Stimulus / Fault:

Observation:

Measured:
- elapsed:
- changed:
- failed:
- unreachable:
- manual intervention:

PASS / FAIL / BLOCKED Basis:

Evidence:
- Issue / PR:
- 11 Runbook Section:
- Command / Runner:
- Before Snapshot:
- After Snapshot:
- Consumer:
- Metric:
- Timeline:

Failure / Block Reason:

Manual Steps:

Cleanup / Rollback:

Follow-up:
```

Cross-role Evidence를 재사용하는 경우:

```text
Dependency Evidence:
- Owner:
- Test / Issue / PR:
- Revision:
- Timestamp:
- Reused Result:
- This Test Boundary:
```

---

## 15. MVP Acceptance

최종 프로젝트 Acceptance는 Network / Ansible / External Infra 한 역할의 12 문서만으로 완료되지 않는다.

본 역할은 Network·External Endpoint·Ansible Execution Evidence를 제공하고, Application·Database·Delivery/Observability 결과와 함께 최종 Gate를 구성한다.

### 15.1 현재 본 역할 검증 상태

| 항목 | 현재 상태 | 대표 Evidence |
| --- | --- | --- |
| Project 고정 실행환경 | PASS — 기존 검증 범위 | Infra #84 / #94 / PR #96 |
| Inventory / 기존 VM 접근 | PASS — 기존 Smoke 범위, 실행 시 재확인 | Infra #84 |
| VRouter NIC / Forwarding / Route | PASS — 과거 Actual Run Evidence | Infra PR #103 / #134 |
| VRouter 현재 Safe Runner 재실행 | BLOCKED / 재정합 필요 | 현재 Safe Runner Source |
| VRouter 정상 2회차 | PASS — 과거 Revision Evidence | Infra PR #134 |
| Route Drift 복원 | PASS — 과거 Revision Evidence | Infra PR #134 추가 검증 |
| Firewall zone / Policy / NAT | Partial | `vrouter_firewall` Source / PR #206 일부 경로 |
| Worker→VRouter `9100/tcp` | PASS — 과거 Cross-VRouter Runtime Evidence 범위 | Infra #205 / PR #206 |
| `lb-01` VIP / Return Route | Partial — 현재 Runner 재검증 필요 | `lb_network` Source |
| HAProxy Listener / Backend | Partial | `lb_haproxy` Source / Consumer Evidence |
| Game External Endpoint | PASS — Cross-role 확인 범위 | Infra #188 / KAI Evidence |
| DB VIP / FQDN Consumer | PASS — Cross-role 확인 범위 | Database / Storage / Recovery Evidence |
| Kubernetes API IP Listener | Implemented — Listener 경로 존재 | `lb_haproxy` Source |
| Kubernetes API FQDN | Not Tested / `pending_api_san` | `project_endpoints` |
| VRouter Metric Target | PASS — 과거 Target Evidence | Infra PR #206 / DOB-MET-02 |
| HAProxy Metric Target | PASS — 과거 Target Evidence | Infra PR #208 / DOB-MET-02 |
| Chrony | PASS — 과거 검증 대상 | Infra #39 / PR #40 |
| Safe Runner 정책 | Partial | Infra #160 / PR #184 |
| Safe Runner 4개 주요 Playbook 현재 Apply | Not Tested / Blocked 혼재 | 현재 Runner Source |
| SSH Bootstrap #27 | Planned | Infra #27 Open |
| 신규 VM 전체 Clean Build | Not Tested | Evidence 없음 |
| `lb-02` / Keepalived | Deferred | 1차 MVP 축소 범위 |
| M-05 종합 자동화 검증 | Partial | VRouter 일부 Evidence 존재 |
| M-05 수동 대비 정량 개선률 | Not Tested | 동일 조건 Before/After 없음 |
| P4 전체 Acceptance | 본 역할 단독 PASS 금지 | Cross-role |

### 15.2 최종 본 역할 Acceptance에 필요한 남은 검증

현재 문서 기준으로 남은 핵심은 다음과 같다.

- `vrouter_network` Safe Runner Evidence 정합화 후 현재 Revision Actual Run
- `vrouter_firewall` 자동 Check Mode·Actual Apply 경계 확인
- `lb_network` 전체 Check Mode·Actual Apply 확인
- `lb_haproxy` Check Mode 조회/Assert 정합성과 Actual Apply 확인
- 위 Playbook의 현재 Revision 정상 2회차 Evidence
- 필요한 경우 대표 Drift 복원 Run
- Network / Ansible 대표 Role을 동일 Revision에서 묶은 `NAE-AUT-01` 종합 Run
- Kubernetes API FQDN SAN·게시가 실제로 진행될 경우 `NAE-END-02`
- SSH Bootstrap #27 완료 시 `NAE-SSH-01`
- 신규 VM 전체 Clean Build 수행 시 `NAE-CLEAN-01`
- M-05의 수동 대비 정량 개선률을 주장하려면 동일 시작 조건의 Before/After 측정

이미 확인된 과거 Runtime Evidence를 다시 미완료로 표현하지 않는다.

반대로 현재 실행하지 않은 항목을 과거 Evidence만으로 현재 PASS로 승격하지 않는다.

### 15.3 Deferred / Planned 항목

다음은 현재 1차 MVP 문서 작성 완료를 막는 필수 조건으로 취급하지 않는다.

- `lb-02`
- Keepalived 자동 VIP 전환
- SSH Bootstrap #27의 현재 Planned 상태
- Kubernetes API FQDN을 사용하지 않는 현재 Consumer 경로
- 전체 Clean Build 미실행

다만 해당 기능을 “완료” 또는 “검증 완료”라고 표현해서는 안 된다.

---

## 16. 문서 완료 기준

본 문서는 다음 조건을 모두 만족할 때 작성 완료로 본다.

- 02의 M-01~M-05와 역할 관계가 추적된다.
- M-05 Ansible 개선의 직접 책임이 명확하다.
- 09 Gate A~G와 Test Case가 연결된다.
- Test Traceability에서 각 Test가 11 Runbook의 관련 절과 연결된다.
- 11의 실행 명령을 과도하게 반복하지 않고 Test Contract로 연결한다.
- 각 주요 Test에 Preconditions가 있다.
- 각 주요 Test에 Stimulus 또는 Fault가 있다.
- 각 주요 Test에 Observation / Measurement가 있다.
- 각 주요 Test에 PASS/FAIL/BLOCKED 판정 기준이 있다.
- 각 주요 Test에 Evidence 출처가 있다.
- PASS 또는 Validated 결과에는 실제 Issue·PR·Runtime Evidence를 연결한다.
- Runtime과 Persistent 설정을 구분한다.
- 정상 2회차와 Drift 복원을 구분한다.
- Firewall Network Boundary와 Prometheus Consumer 결과를 구분한다.
- Git Revert와 Runtime Rollback을 동일한 절차로 취급하지 않는다.
- `--inspect-only`와 Actual Apply를 구분한다.
- Safe Runner에서 정상 경로의 PASS와 정책상 BLOCKED를 같은 결과로 취급하지 않는다.
- Safe Runner 차단을 직접 실행으로 우회하지 않는다.
- 과거 Runtime Evidence와 현재 Revision 재실행 가능성을 구분한다.
- VRouter Metric과 HAProxy Metric을 구분한다.
- VRouter Metric 주소는 사설 Gateway `192.168.51.10:9100` ~ `192.168.54.10:9100` 기준으로 기록한다.
- HAProxy Metric은 `10.1.93.78:8404`로 별도 기록한다.
- `node-exporter-external 7/7`의 대상 구성을 잘못 확대하지 않는다.
- `lb-01` VIP는 Address와 Prefix를 함께 검증한다.
- Return Route는 목적지와 Next Hop을 함께 검증한다.
- Firewall의 선택적 Masquerade와 사설망 NAT 제외 경계를 정확하게 기록한다.
- 미실행·차단·보류·계획 항목을 PASS처럼 작성하지 않는다.
- `lb-02`·Keepalived를 현재 구현으로 표현하지 않는다.
- SSH Bootstrap #27을 완료로 표현하지 않는다.
- Kubernetes API FQDN을 현재 검증 완료로 표현하지 않는다.
- 신규 VM 전체 Clean Build를 완료로 표현하지 않는다.
- Cross-role Evidence의 Owner·Source·Revision 또는 시점을 남긴다.
- Secret·Password·Token·Private Key를 Evidence에 남기지 않는다.
- 실패 Run을 보존하고 재실행 시 새 Run ID를 사용한다.
- 문서 작성 완료와 Network / Ansible / External Infra 전체 Acceptance를 별도로 판정한다.
- 실제 Repository·Runtime Evidence와 재대조 후 추가 필수 보완사항이 없을 때 종료한다.
