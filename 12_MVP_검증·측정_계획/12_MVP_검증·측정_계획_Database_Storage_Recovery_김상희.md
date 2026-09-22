# MVP 검증·측정 계획
## Database / Storage / Recovery

## 1. 목적과 문서 경계

이 문서는 「石나가는 판단」 1차 프로젝트의 Database / Storage / Recovery 영역에서 **무엇을 시험하고, 무엇을 측정하며, 어떤 근거로 PASS/FAIL을 판정할지** 정의한다.

```text
09 = What / Why / Responsibility / Gate(A~G)
11 = Pre-check / Apply / Verify / Re-run / Recovery / Rollback
12 = Test Case / Preconditions / Stimulus·Fault / Measurement / PASS·FAIL / Evidence
```

12는 11의 실행 명령을 복제하지 않는다. 실제 실행 절차는 11을 참조하고, 본 문서는 각 Test의 선행조건·실행 자극/장애·관찰 항목·PASS/FAIL 기준·측정 Evidence를 정의한다.

직접 기준:

- [`02_石나가는_판단_핵심_문제_및_검증_목표.docx`](../01-08_기획·설계_Baseline/02_石나가는_판단_핵심_문제_및_검증_목표.docx)
- [`09_MVP_실행·통합_실시설계_Database_Storage_Recovery_김상희.md`](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Database_Storage_Recovery_김상희.md)
- [`11_MVP_구축·자동화_Runbook_Database_Storage_Recovery_김상희.md`](../11_MVP_구축·자동화_Runbook/11_MVP_구축·자동화_Runbook_Database_Storage_Recovery_김상희.md)
- [`PROJECT_CHANGES.md`](../PROJECT_CHANGES.md)
- `seokpan-infra` 최신 `main`, Issue #113/#115/#143/#156/#176/#194/#199/#211, 관련 PR(#191/#197/#200/#201 포함), Runtime/Recovery Evidence
- `F_redis_recovery_contract.md`, `E_infra_snapshot.md`(9-9~9-11절), `H_progress_checklist.md`(F-9/F-10/I절)

실제 Test Run 직전에는 `seokpan-infra`의 최신 Revision과 열린 Issue/PR을 다시 확인한다.

---

## 2. 상위 검증축과 Gate 대응

02의 검증축을 대체하지 않는다. 본 역할은 M-04(DR)를 핵심으로 소유하며, M-01/M-03에는 Data Consumer 관점에서 Cross-role로 연결된다.

| ID | 검증축 | 본 역할과의 관계 |
| --- | --- | --- |
| M-01 | 동시성 정확성 | Cross-role — MariaDB/Redis가 실제로 정확한 상태를 저장하는지 Data 관점에서 연결 |
| M-02 | Vote 처리 성능 | Cross-role — DB/Redis 부하 시 지연·자원 사용을 병목 진단에 제공 |
| M-03 | Backend 장애 복구 | Cross-role — Backend 장애 시 DB/Redis 상태 일관성 유지 여부 |
| M-04 | DR | **핵심** — MariaDB Backup/Recovery(DR-01), etcd Snapshot/Restore(DR-02), Redis DR |
| M-05 | Ansible 개선 | 핵심 — Backup/Recovery/Exporter Role의 구축·재구축 시간, 개입, 실패 Task |

09 Gate와의 대응:

```text
Gate A (Database Provider Ready)      → DSR-DB-01
Gate B (Database Security Contract)   → DSR-SEC-01
Gate C (NFS / Storage Provider Ready) → DSR-NFS-01
Gate D (MariaDB Backup Ready)         → DSR-BAK-01
Gate E (MariaDB Recovery Ready)       → DSR-REC-01 (M-04 DR-01)
Gate F (Redis Persistence / Recovery) → DSR-REDIS-01 / DSR-REDIS-02
Gate G (Kubernetes etcd DR)           → DSR-ETCD-01 (M-04 DR-02)
```

추가 통합 Evidence:

```text
Vault ↔ Kubernetes Secret ↔ MariaDB 계정 정합성
Backup Chain 공유 상태(NFS) / Shared Lock
DB/NFS Observability Exporter(node_exporter / mysqld_exporter / maxscale_exporter)
Application Data Consumer 연결(Backend → MaxScale → MariaDB, Backend → Redis)
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
In Progress
Blocked
Deferred
Not Tested
Failed
```

다음을 같은 의미로 취급하지 않는다.

```text
Backup 파일 존재 ≠ Backup Chain 정상 ≠ Restore 가능 ≠ Recovery 완료
Database Ready ≠ Backup Ready ≠ Recovery Ready ≠ Application Consumer Ready
Snapshot 생성 ≠ Snapshot 검증 ≠ 격리 환경 Restore 성공 ≠ RTO 측정 완료
```

### 3.2 임의 목표값 금지

MariaDB DR-01(RTO/RPO)과 etcd DR-02(RTO)는 이미 실측 완료된 고정 Evidence다(각각 이슈 #194 2026-09-18 close, PR #191 2026-09-16 merged). 다음은 이 고정 Evidence를 실험/실측 근거 없이 다른 값으로 임의 변경하지 않는다는 뜻이며, 동시에 이 값들을 "미확정"으로 되돌려 표현하지도 않는다.

- MariaDB DR-01: Isolated RTO=27.376초, Production RPO=3건/RTO 1분29초(단일)·4분(이중화) — 확정값, Chain 구성·서버 사양이 바뀌면만 재측정
- etcd DR-02: 자동화 기준 RTO=52초(#176/PR#191) — 확정값. 자동화 이전 수동 1차 측정값(38분44초, #156)은 별도 참고값으로 병기하며 서로 대체하지 않는다
- Redis DR-03: RTO/RPO 목표값은 아직 정의되지 않았다(자동화 자체가 미착수이므로) — 이 항목만 임의 목표값 금지 원칙이 적용된다
- Backup Retention 일수 변경 근거
- Exporter Alert Rule의 임계값(2차 프로젝트 항목)

etcd DR-02 RTO 52초는 K8s 상태 복구 시간만을 의미하며 전체 클러스터 재구축 시간은 포함하지 않는다는 측정 범위를 벗어난 해석은 금지한다.

### 3.3 현재 MVP 범위

1차 프로젝트에서는 etcd DR 스코프를 "K8s 서버는 살아있고 클러스터 내부 상태 데이터만 손상된 경우"로 한정한다. 서버 자체 손상 시 신규 서버로 전체 이관 복구(이슈 #178)는 본 MVP PASS 조건에 포함하지 않는다.

Redis DR-03(PVC 손상 시 MariaDB 기준 재구성)은 격리 환경에서 1차 검증까지는 완료했으나(이슈 #115, 2026-09-18), 운영 Redis에 대한 Ansible 자동화·쓰기 반영 권한 결정은 1차 프로젝트 범위에서 Deferred로 관리하며 완료 조건에 포함하지 않는다. mysqld_exporter Alert Rule과 maxscale_exporter 실배포도 동일하게 Deferred다.

---

## 4. Evidence 규격

각 Run 최소 Metadata:

```text
Test Case ID
Run ID
Target — mariadb / maxscale / nfs / redis / etcd / exporter
Start / End Timestamp
Operator
Infra Commit SHA
Ansible Branch
현재 Master/Replica 배치(maxctrl list servers 결과 요약)
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
명령 출력(Secret 제외)
Backup Chain 상태 파일(`/mnt/nfs-db-backup/.state/.backup_chain_state.json`) 스냅샷
SHA-256 체크섬
GTID / Replication 상태
etcd Snapshot Status(Hash/Revision/Total Keys)
Quorum / kube-apiserver 확인 결과
Recovery Timeline(단계별 시각)
RTO / RPO 값
```

금지:

```text
Password
전체 DB URL
Vault 평문 값
Token
kubeconfig Credential
Kubernetes/Vault Secret 실제 값
```

### 4.1 Run ID

```text
<TEST-CASE-ID>-YYYYMMDD-HHMM-<short-sha>
```

예:

```text
DSR-REC-01-20260916-1400-a1b2c3d
```

### 4.2 실패 Run 보존

```text
실패 Run 보존
→ 원인·조치 기록
→ 새 Run ID
→ 재실행
```

FAIL을 수정해서 PASS로 덮어쓰지 않는다. MariaDB DR-01의 초기 측정값을 삭제하지 않고 "최종 값 아님"으로 표시한 채 보존한다.

---

## 5. 정량 측정 Catalog

| 항목 | 단위 | 측정 방법/출처 | 용도 |
| --- | --- | --- | --- |
| Backup 소요 시간 | s | Backup 시작~완료 Timestamp | M-05, Recovery 계획 |
| Backup Chain 유효 판정 정확도 | PASS/FAIL count | `.backup_chain_state.json` 대조 | Gate D |
| Backup 무결성 | SHA-256 일치 여부 | `sha256sum` | Gate D |
| Recovery 소요 시간(단계별) | s | Full Prepare/Incremental Prepare/기동/Replication/서비스 확인 각 Timestamp | M-04 RTO |
| RTO(MariaDB) | s | 장애 인지~서비스 접근 가능 확인 | M-04, Gate E |
| RPO(MariaDB) | s | 장애 시점 - 마지막 유효 Backup 시점 | M-04, Gate E |
| Replication 재구성 성공률 | % | 성공 재구성 / 시도 횟수 | Gate E |
| GTID 정합성 | 일치/불일치 | `SHOW SLAVE STATUS` 비교 | Gate E |
| Redis Write/Read 성공률 | % | 성공 요청 / 전체 요청 | Gate F |
| Redis Pod 재생성 후 데이터 보존율 | % | 재생성 전/후 Key 비교 | Gate F |
| etcd Snapshot 생성 시간 | s | `snapshot save` 시작~완료 | Gate G |
| etcd Restore RTO | s | Restore 시작~Quorum+Object 비교 완료 | M-04, Gate G(고정 Evidence 52초) |
| etcd Snapshot Hash 일치 | 일치/불일치 | `etcdutl snapshot status` | Gate G |
| Kubernetes Object 수 일치 | count 비교 | Namespace/Deployment/Secret 원본 대비 복구본 | Gate G |
| node_exporter idempotency | changed count | 2회차 `--check --diff` 결과 | M-05 |
| mysqld_exporter 계정 권한 범위 | GRANT 목록 | `SHOW GRANTS` | Observability |
| Ansible 재구축 소요 시간 | s/min | Playbook 실행 시작~완료 | M-05 |
| Ansible 실패 Task 수 | count | `--check` 및 실제 실행 결과 | M-05 |

측정 출처가 여러 개인 경우 Run Metadata에 실제 Source를 명시한다.

---

## 6. Dependency Evidence 재사용

Revision이 바뀌지 않은 검증 완료 Capability는 재사용할 수 있다.

예:

- MariaDB 2대 + MaxScale 기본 구성(Gate A)
- Vault ↔ Kubernetes Secret ↔ DB 계정 정합성(Gate B, Infra #172)
- NFS Server / Kubernetes NFS Provisioner(Gate C, Infra #52/PR #61/#78, GitOps #10/#13)
- Backup Chain 공유 상태/Shared Lock 구조(Infra #166/#167, PR #168/#171)
- etcd DR-02 격리 환경 Restore 절차(Infra PR #191)
- node_exporter 배포(Infra #199)

재검증 조건:

```text
Ansible Role/Playbook 변경
MariaDB/MaxScale 버전 변경
Vault Credential 변경
NFS 서버/Export 구성 변경
etcd 클러스터 재구축
장애·복구 후
Evidence Revision 불일치
```

특히 etcd DR-02의 52초 RTO는 특정 실측 Run의 결과이며, 클러스터 구성이나 도구 버전이 바뀌면 재측정 없이 그대로 재사용하지 않는다.

---

## 7. 현재 검증 기준 상태

| 영역 | 상태 | 12 판정 |
| --- | --- | --- |
| MariaDB / MaxScale 기본 구성(Gate A) | Validated | 재사용 가능 |
| Vault/Secret/DB 계정 정합성(Gate B) | Validated | 재사용 가능 |
| MariaDB TLS(Gate B) | Validated | 재사용 가능 |
| NFS / Kubernetes Storage(Gate C) | Validated | 재사용 가능 |
| MariaDB Backup Chain(Gate D) | Validated | PASS |
| Backup 공유 상태 / Shared Lock(Gate D) | Validated | PASS |
| MariaDB Recovery 절차 자체(Gate E) | Validated | PASS(절차) |
| MariaDB DR-01 최종 RTO/RPO(Gate E) | **Validated(완료, 이슈 #194 2026-09-18 close)** | PASS — Isolated RTO 27.376초 / Production RPO 3건·RTO 1분29초(단일)·4분(이중화) |
| Redis Persistence(Gate F) | Validated | PASS |
| Redis DR-03(PVC 손상 시 MariaDB 기준 재구성, Gate F) | **Validated(1차 Infra 레벨, 격리 환경, 2026-09-18)** | PASS(검증 범위 한정) — 운영 자동화는 이슈 #115 open 상태로 잔여 |
| etcd Snapshot 생성/무결성/NFS 전송(Gate G) | Validated(축소 스코프) | PASS — 이슈 #113(2026-09-08 스코프 축소 후 completed) |
| etcd DR-02 E2E 수동(Gate G) | Validated | PASS, RTO 38분44초(트러블슈팅 포함) — 이슈 #156 |
| etcd DR-02 E2E 자동화(Gate G) | Validated | PASS, RTO 52초 — 이슈 #176, PR #191(2026-09-16 merged) |
| node_exporter(4대) | Validated | PASS |
| mysqld_exporter 수집 | Validated(수집까지) | PASS for 수집 경로(TS-043 SLAVE MONITOR 권한 버그 수정 완료), Alert Rule은 범위 밖 |
| maxscale_exporter | Deferred | Not Tested, 2차 프로젝트 |
| 레거시 cron orphan 코드 정리 | Not Done | Not Tested |
| Application Data Consumer(Backend→DB/Redis) | Cross-role Validated(A-10 기준) | 재사용 가능, 변경 시 재검증 |

DR-01/DR-02(etcd)는 실측까지 종결됐지만, Redis DR-03의 "검증 완료"와 "운영 자동화 완료"는 다른 의미이며, A-10 Runtime Integration 완료와 전체 Data Platform DR Acceptance 완료도 같은 의미로 사용하지 않는다.

## 8. Test Traceability Matrix

| Test Case | 목적 | Gate | 11 | 현재 상태 |
| --- | --- | --- | --- | --- |
| DSR-PRE-01 | Master/Replica 및 Dependency Snapshot | 공통 | 4 | Validated |
| DSR-DB-01 | Database Provider Ready | A | 4.3 | PASS |
| DSR-SEC-01 | Vault/Secret/TLS 정합성 | B | 4.6 | PASS |
| DSR-NFS-01 | NFS / Kubernetes Storage | C | 5 | PASS |
| DSR-BAK-01 | MariaDB Backup Chain | D | 6 | PASS |
| DSR-BAK-02 | Backup 동시 실행/Lock | D | 6.5 | PASS |
| DSR-REC-01 | MariaDB Recovery RTO/RPO(DR-01) | E | 7.5 | **PASS(완료, 이슈 #194)** |
| DSR-REC-02 | Master 경로 버그 수정 재검증(PR #197) | E | 7.6 | PASS |
| DSR-REDIS-01 | Redis Persistence(Pod 재시작) | F | 8.2 | PASS |
| DSR-REDIS-02 | Redis DR-03(PVC 손상 시 MariaDB 기준 재구성) | F | 8.3~8.5 | **PASS(1차 Infra 레벨, 격리 환경)** |
| DSR-REDIS-03 | Redis DR-03 운영 자동화 | F | 8.6 | Not Tested / Deferred(BLOCKING: RBAC 결정) |
| DSR-ETCD-01 | etcd Snapshot 생성/무결성/전송 | G | 9.1~9.5 | PASS(축소 스코프, #113) |
| DSR-ETCD-02 | etcd DR-02 E2E 수동(#156) | G | 9 참고값 | PASS, RTO 38분44초 |
| DSR-ETCD-03 | etcd DR-02 E2E 자동화(#176/PR#191) | G | 9.6~9.7 | PASS, RTO 52초 |
| DSR-OBS-01 | node_exporter | Cross | 10.1 | PASS |
| DSR-OBS-02 | mysqld_exporter | Cross | 10.2 | PASS(수집), Alert 범위 밖 |
| DSR-OBS-03 | maxscale_exporter | Cross | 10.3 | Deferred |
| DSR-CFG-01 | cron 마커/orphan 정리 | D | 6.7 | Not Tested |
| DSR-DAT-01 | Application Data Consumer 연결 | A~F | 09 §4 | Cross-role Validated(A-10 기준) |

`PASS`는 해당 Test Case의 현재 정의 범위에 실제 실측 Evidence가 존재할 때만 사용한다.

`DSR-REC-01`은 이슈 #194(2026-09-18 close)에서 Isolated/Production 두 시나리오 모두 실측을 완료했으므로 PASS로 판정한다. `DSR-REDIS-02`(1차 Infra 레벨 검증)와 `DSR-REDIS-03`(운영 자동화)은 서로 다른 Test Case이므로 전자의 PASS가 후자의 PASS를 의미하지 않는다.

## 9. Test Contract Matrix

| Test | Preconditions | Stimulus / Fault | Observation | PASS 핵심 | Evidence |
| --- | --- | --- | --- | --- | --- |
| DSR-PRE-01 | 대상 Revision 식별 | 없음, Snapshot | Master/Replica, NFS mount, Vault | Dependency와 Revision 일치 | Snapshot/Commit |
| DSR-DB-01 | MariaDB/MaxScale 배포 | 없음, 상태 조회 | `maxctrl list servers`, Schema/계정 | 서버 2대+MaxScale 정상, 계정 분리 확인 | 명령 출력 |
| DSR-SEC-01 | Vault/Secret 존재 | Key 이름 대조 | Vault↔Secret↔DB 계정 | Key 일치, TLS 연결 성공 | Key 목록/TLS 로그 |
| DSR-NFS-01 | NFS/StorageClass 배포 | Pod 재생성 | PVC bound, 데이터 유지 | 재생성 후 데이터 보존 | PVC/Pod 로그 |
| DSR-BAK-01 | Replica 확인, Lock 없음 | Full/Incremental 실행 | Chain 상태, SHA-256 | Chain 갱신, 무결성 일치 | 상태 파일/체크섬 |
| DSR-BAK-02 | 두 호스트 동시 실행 조건 | 동시 실행 시도 | Lock 획득 로그 | 동시 획득 0건 | Lock 로그 |
| DSR-REC-01 | 유효 Backup Chain, Isolated/Production 모드 구분 | Recovery 실행(양쪽 시나리오) | 단계별 Timestamp, 데이터/Replication/MaxScale | Isolated RTO, Production RPO/RTO 실측 완료 | Timeline/GTID/로그(이슈 #194) |
| DSR-REC-02 | PR #197 반영 확인 | Master/Replica 각각 실행 | `failed` 카운트 | 양쪽 `failed=0` | 실행 로그 |
| DSR-REDIS-01 | Redis Runtime | Write→Pod 재생성→Read | Key 값 비교 | 값 유지 | redis-cli 출력 |
| DSR-REDIS-02 | 격리 StatefulSet(redis-dr03-test), MariaDB synthetic 데이터 | AOF 손상 재현, MariaDB 기준 재구성 | 재구성값 vs 사전 계산값, Ahead/Behind/Exact 판정 | 값 일치, Loss/Duplicate/Stale 0건 | 이슈 #115 코멘트, `F_redis_recovery_contract.md` |
| DSR-REDIS-03 | DSR-REDIS-02 PASS, RBAC 결정 | (미정의 — 자동화 착수 전) | (미정의) | (미정의 — Deferred, BLOCKING: 운영 쓰기 권한) | — |
| DSR-ETCD-01 | 클러스터 Health, 도구 SHA 검증 | Snapshot→NFS 전송(/srv/nfs/etcd-dr) | Hash/Revision/SHA-256 | 원본과 일치 | Snapshot Status(이슈 #113) |
| DSR-ETCD-02 | DSR-ETCD-01 PASS, 별도 물리PC 격리망 | 수동 3-member Restore→API→Object | Quorum/Object 수/RTO | 원본과 일치, RTO 기록 | 이슈 #156 코멘트 |
| DSR-ETCD-03 | DSR-ETCD-01 PASS, Safety Guard/Restore Decision Gate | 자동화 Playbook 실행 | Quorum/Object 수/RTO(자동 기록) | 원본과 일치, Object diff 0 | PR #191, dr-evidence/ |
| DSR-OBS-01 | Role 배포 | 재실행(`--check --diff`) | changed count | 2회차 changed=0 | ansible 출력 |
| DSR-OBS-02 | exporter_svc 계정 배포(`SLAVE MONITOR` 포함) | Metric 수집 확인 | GRANT 목록, Prometheus 수집 여부 | 필요 권한 보유, 수집 확인 | GRANT 출력/Prometheus, TS-043 |
| DSR-OBS-03 | (Deferred) | — | — | — | — |
| DSR-CFG-01 | 스케줄 변경 이력 | `crontab -l` 조회 | orphan 항목 존재 여부 | orphan 0건(코드 정리 완료 시) | crontab 출력 |
| DSR-DAT-01 | Backend Runtime | 실제 요청 | DB/Redis 실제 데이터 반영 | Application 흐름에서 정상 반영 | Backend 로그/DB 조회 |

---

## 10. Test 상세 판정

### DSR-DB-01 — Database Provider Ready

Preconditions: MariaDB 2대, MaxScale 1대 배포 완료.

PASS:

- `maxctrl list servers`에서 Master 1, Slave 1, 모두 `Running`.
- `stone_game` Database 및 7개 테이블 존재.
- 6개 계정(identity_svc/game_svc/db_admin/backup_svc/repl_user/maxscale_monitor) 및 `exporter_svc` 존재.

현재 결과: `PASS`.

### DSR-SEC-01 — Security Contract

PASS:

- Vault Key와 Kubernetes Secret Key 이름 일치.
- MariaDB TLS 1.3(TLS_AES_256_GCM_SHA384) handshake 성공.
- SAN에 `db.seokpan.soldesk.store` 포함.
- 재발급 시 root:root 초기화 문제가 없음(`configure.yml` 권한 보정 태스크 적용 확인).

현재 결과: `PASS`. Evidence: PR #137.

### DSR-NFS-01 — NFS / Storage

PASS:

- `showmount -e`에 `/srv/nfs/k8s`, `/srv/nfs/db-backup` 노출.
- PVC가 StorageClass를 통해 동적 생성.
- Pod 재생성 후 PVC 데이터 유지.

현재 결과: `PASS`.

### DSR-BAK-01 — MariaDB Backup Chain

PASS:

- Full/Incremental Backup 정상 생성.
- `.backup_chain_state.json`의 `chain_origin`이 실제 실행과 일치.
- SHA-256 무결성 일치.
- 유효 Chain 없을 때 Full로 자동 승격(이슈 #129 로직) 확인.
- 7일 Retention 정책대로 오래된 백업 정리.

현재 결과: `PASS`. Evidence: PR #117/#125/#131.

### DSR-BAK-02 — Backup 동시 실행 보호

Stimulus: 양쪽 호스트에서 거의 동시에 Backup 실행 시도(3회 반복).

PASS: 동시 획득 0건, `flock` 기반 직렬화 확인, `lock_wait_seconds=60`으로 NFSv4 콜백 폴링 주기(약 30초)를 흡수.

현재 결과: `PASS`. Evidence: Infra #166 PR #168.

### DSR-REC-01 — MariaDB Recovery RTO/RPO(DR-01)

Preconditions: DSR-BAK-01 PASS, `backup_restore_mode`(isolated/production) 명시적 구분.

Stimulus 및 결과는 두 시나리오로 나뉜다.

**Isolated 모드**: `mariadb_restore_chain.yml`, mariadb-02 대상, `chain_20260916`(Full 1개) → **RTO = 27.376초**, Count/CHECKSUM/FK Gate 7개 테이블·7개 관계 전부 PASS.

**Production 모드**(양쪽 동시 유실 최악 시나리오): 더미데이터 A 삽입 → Incremental 백업 → 더미데이터 B 삽입(미백업) → mariadb-01/02 양쪽 동시 정지 → `mariadb_dr_recovery.yml` 4단계를 master→replica 순 완주 → **RPO = 3건 손실**, **RTO(단일 노드 재개) = 1분 29초**, **RTO(양쪽 이중화 완전 정상화) = 4분 0초**. Split-brain 방지 안전장치(read_only 강제 배포)의 최초 실측 검증도 이때 완료(재편입 시점 자동 해제 확인).

PASS 기준: 두 시나리오 모두 복구 데이터 건수·무결성·외래키·GTID·Replication·MaxScale·서비스 접근이 정상이고 RTO/RPO가 실제 측정값으로 기록됨.

현재 결과: `PASS`(완료, 2026-09-18 이슈 #194 close). 위 수치가 최종 확정값이며, Chain 구성이나 서버 사양이 달라지면 새 Run ID로 재측정한다(기존 값을 덮어쓰지 않음).

Evidence: Infra #194(코멘트에 단계별 로그·수치 상세), PR #197(Master 경로 fatal 버그 수정).

### DSR-REC-02 — Master 경로 버그 수정 재검증

Stimulus: DSR-REC-01의 Production 모드 실행이 Master 경로, Replica 경로를 순서대로 통과.

PASS: 두 경로 모두 `failed=0`(`replication_setup.yml:117` Gate 태스크의 `when` 조건 수정 반영 확인).

현재 결과: `PASS`. Evidence: PR #197(리뷰어 ggbun2 승인, TS-042).

### DSR-REDIS-01 — Redis Persistence

Stimulus: Key Write → Pod 삭제(재생성 유도) → 동일 Key Read.

PASS: 값 유지, PVC bound 유지.

현재 결과: `PASS`. Evidence: seokpan-gitops#7.

### DSR-REDIS-02 — Redis DR-03(PVC 손상 시 MariaDB 기준 재구성)

Preconditions: 격리 StatefulSet `storage-infra/redis-dr03-test`(운영과 동일 스펙: redis:8.10.1, appendonly yes/everysec, nfs-k8s PVC), MariaDB synthetic 데이터 적재.

Stimulus: AOF(base.rdb) 직접 손상 → CrashLoopBackOff 재현 → MariaDB `move` 테이블(`move_no` 순) 기준 SQL 수동 재구성 → Redis Ahead/Behind/Exact 3케이스 인위 재현.

Observation: 재구성된 move_no/turn_no/current_team/last_move 값과 사전 계산값 비교, Ahead(무효화)/Behind(재동기화) 처리 결과, Data Loss/Duplicate/Stale 여부.

PASS: 재구성값이 사전 계산값과 완전 일치, 3케이스 판정이 계약(`F_redis_recovery_contract.md`)대로 정확히 동작, Loss/Duplicate/Stale 전부 0건.

현재 결과: `PASS`(1차 Infra 레벨, 격리 환경 한정, 2026-09-18). 이슈 #115는 이 결과와 별개로 open 상태를 유지한다 — 이유는 검증 미완료가 아니라 DSR-REDIS-03(운영 자동화)이 아직 남아 있기 때문.

Evidence: 이슈 #115 코멘트(절차 확정 + 최종 Evidence), `F_redis_recovery_contract.md` 갱신분.

### DSR-REDIS-03 — Redis DR-03 운영 자동화

현재 `Not Tested / Deferred`. Ansible 자동화(`redis-dr-recovery-automation-handoff.md`로 인계)와 운영 `platform/redis-0`에 대한 재구성값 쓰기 반영 권한 결정이 BLOCKING 선행조건이다. 관찰/장애주입용 권한(`pods:delete`, `pods/log:get`)은 `platform/ksh` SA에 이미 부여됐으나(seokpan-gitops#47/#48), `pods/exec`는 부여하지 않기로 확정되어 있어 별도의 쓰기 경로 설계가 필요하다. 이 항목을 1차 프로젝트 MVP Acceptance의 필수 조건으로 포함하지 않는다.

### DSR-ETCD-01 — etcd Snapshot 생성/무결성/전송(축소 스코프)

Preconditions: 3-member 클러스터 Health, 도구(`/opt/seokpan/etcd-tools/current/`) SHA-256 검증.

Stimulus: Snapshot 생성(cp-01) → HASH/SHA-256 무결성 검증 → NFS 전용 경로(`/srv/nfs/etcd-dr`) 전송 및 재검증.

PASS: Snapshot Hash/Revision/Total Keys 기록, 전송 후 SHA-256 재검증 일치.

현재 결과: `PASS`(2026-09-08 스코프 축소 후 completed). Evidence: 이슈 #113. 격리 3-member Restore·API/Object 검증·RTO 측정은 이 Test Case의 범위 밖이며 DSR-ETCD-02/03이 담당한다.

### DSR-ETCD-02 — etcd DR-02 E2E 수동 검증

Preconditions: DSR-ETCD-01 PASS, 별도 물리PC 격리망(loadgen/loadgen2/loadgen3, VMnet 192.168.55.0/24) 구성.

Stimulus: 신규 Snapshot 생성 → 3-member 수동 Restore → kube-apiserver 단독 기동(self-signed CA client cert) → Namespace/Deployment/Secret Object 비교.

Observation: Namespace 12→12, Deployment 21→21, Secret 37→37, Object diff 0, Quorum/Leader 상태.

PASS: 원본과 복구본 Object 수 완전 일치, Quorum 성립, kube-apiserver 인증 성공.

현재 결과: `PASS`, **RTO = 38분 44초**(트러블슈팅 대응 시간 포함, 2026-09-11 17:37:58~18:16:42). Evidence: 이슈 #156. 이 RTO는 자동화 이전 수동 절차의 참고값이며, MVP Acceptance의 대표 RTO로는 DSR-ETCD-03의 값을 사용한다.

### DSR-ETCD-03 — etcd DR-02 E2E 자동화(Safety Guard/Restore Decision Gate)

Preconditions: DSR-ETCD-01 PASS, Safety Guard(`config_guard.yml`)/Restore Decision Gate/Final Restore Gate 구현.

Stimulus: 자동화 Playbook 실행 — Snapshot 준비→무결성 검증→Safety Guard→Restore Decision→Final Restore Gate→3-member Restore→Quorum/Health→kube-apiserver→API 인증→Object 검증→RTO/Evidence 자동 생성. STOP 경로(`dr_restore_requested=false`)도 별도 검증.

Observation: Namespace 13→13, Deployment 21→21, Secret 39→39, Object diff 0, 단계별 RTO(Restore→Quorum/Quorum→API/API→Object).

PASS: 원본과 복구본 Object 완전 일치, Quorum 성립, RTO가 자동 기록됨, STOP 경로 정상 동작.

현재 결과: `PASS`, **RTO = 52초**(Restore→Quorum 21초 / Quorum→API 27초 / API→Object 4초). Evidence: 이슈 #176, PR #191(2026-09-16 01:32 UTC merged, `dr-evidence/<run-id>/` 자동 생성). 09/12 문서에서 "etcd DR-02 RTO"를 대표값으로 인용할 때는 이 52초를 사용한다.

### DSR-OBS-01 — node_exporter

Stimulus: 4대(mariadb-01/02, maxscale-01, nfs) 배포 후 `--check --diff` 재실행.

PASS: 2회차 `changed=0`.

현재 결과: `PASS`. Evidence: Infra #199.

### DSR-OBS-02 — mysqld_exporter

PASS: `exporter_svc` 계정이 쓰기 권한 없이 수집에 필요한 권한(`SLAVE MONITOR` 포함)만 보유, Prometheus Target에서 수집 확인(`mysql_up 1`, `slave_status` collector 정상).

현재 결과: `PASS`(수집 경로). 배포 초기 `SLAVE MONITOR` 권한 누락 버그(TS-043)는 발견·수정 완료. Alert Rule 등록은 2차 프로젝트로 이관되어 본 Test 범위 밖. Evidence: Infra #199/PR #201, Infra #211(2026-09-21 스코프 확정).

### DSR-OBS-03 — maxscale_exporter

현재 Deferred. REST read-only 계정과 Role 골격의 idempotency·재귀 검증(읽기 정상/쓰기 401 차단/서비스 상태 불변)은 통과했으나, 실제 바이너리 배포 및 Prometheus 연동은 2차 프로젝트 범위다. 이번 MVP Acceptance PASS 조건에 포함하지 않는다.

### DSR-CFG-01 — cron 마커/orphan 정리

현재 결과: `Not Tested`. 수동 제거는 완료했으나 코드 레벨(`state: absent`) 정리가 없어 재배포 시 orphan이 재발할 가능성이 있다. 코드 반영 후 `crontab -l` 결과에 구 이름 항목이 없는지 재검증한다.

### DSR-DAT-01 — Application Data Consumer 연결

Cross-role: A-10 Runtime Integration에서 Backend → MaxScale → MariaDB, Backend → Redis 연결이 검증된 것을 근거로 재사용한다. DB/NFS/Backup/Recovery/etcd 자산이 변경되면 이 Cross-role Evidence를 재검증 대상으로 표시한다.

---

## 11. 검증 Phase

### Phase 0 — Provider / Security / Storage

현재 결과: `Completed`.

```text
DSR-DB-01 PASS
→ DSR-SEC-01 PASS
→ DSR-NFS-01 PASS
```

### Phase 1 — Backup

현재 결과: `Completed`.

```text
DSR-BAK-01 PASS
→ DSR-BAK-02 PASS
```

### Phase 2 — Recovery(DR-01)

현재 결과: `Completed`.

```text
DSR-REC-02(Master 경로 버그 수정): PASS
DSR-REC-01(Isolated/Production RTO/RPO): PASS(완료, 이슈 #194)
```

### Phase 3 — Redis / etcd(DR-02)

현재 상태:

```text
DSR-REDIS-01: PASS
DSR-REDIS-02(1차 Infra 레벨 검증): PASS
DSR-REDIS-03(운영 자동화): Not Tested / Deferred(BLOCKING: RBAC 결정)
DSR-ETCD-01(스냅샷/전송): PASS
DSR-ETCD-02(수동 E2E): PASS(RTO 38분44초, 참고값)
DSR-ETCD-03(자동화 E2E): PASS(RTO 52초, 대표값)
```

Redis DR-03의 1차 검증 PASS를 "Gate F 전체 완료"로 확대 해석하지 않는다 — DSR-REDIS-03(운영 자동화)이 남아 있다.

### Phase 4 — Observability

현재 상태:

```text
DSR-OBS-01: PASS
DSR-OBS-02: PASS(수집), Alert 범위 밖
DSR-OBS-03: Deferred
DSR-CFG-01: Not Tested
```

### Absolute Gates

현재 P4/최종 Acceptance에서 여전히 유효한 절대 조건:

```text
Redis DR-03 운영 자동화(DSR-REDIS-03) 미착수, RBAC 결정 미확정
→ Gate F 전체 PASS 금지(Persistence + 1차 DR-03 검증까지만 PASS)

maxscale_exporter 미배포
→ DSR-OBS-03 PASS 금지

cron orphan 코드 정리 미완
→ DSR-CFG-01 PASS 금지
```

---

## 12. Before → Change/Fault → After

### MariaDB Recovery

```text
정상 Runtime Snapshot(장애 전 데이터/Replication 상태)
→ 격리 환경에서 Backup Chain 기반 Restore
→ 복구 후 Snapshot(데이터/Replication/MaxScale 상태)
```

비교: 데이터 건수, 무결성, GTID, Replication 상태, RTO/RPO.

### Backup 동시 실행

```text
단일 호스트 정상 실행 상태
→ 두 호스트 동시 실행 시도(3회)
→ Lock 로그 및 Chain 상태 비교
```

비교: 동시 획득 횟수(목표 0), Chain 오염 여부.

### etcd DR

```text
운영 클러스터 정상 Snapshot
→ 격리 환경 Restore
→ 원본과 복구본 Object 비교
```

비교: Hash/Revision/Object 수 일치, RTO.

---

## 13. 중단 기준

즉시 중단:

- 데이터 손실 위험
- 예상 외 GTID/Replication 상태
- Master 오판(현재 Master 미확인 상태에서 쓰기 작업 시도)
- Secret/Vault 평문 노출
- Backup Chain 상태 파일 손상 의심 상태에서 강행 실행
- etcd Snapshot Hash 불일치 상태에서 Restore 강행
- Recovery를 운영 환경(격리 환경 아님)에 직접 적용하려는 시도

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
Target:
Date/Time:
Operator:
State: PASS / FAIL / BLOCKED / NOT TESTED

Infra Commit:
Ansible Branch:
Master/Replica 배치:

Preconditions:
Stimulus/Fault:
Observation:
Measured:
PASS/FAIL Basis:

Evidence:
- Issue/PR:
- Runbook/Command:
- Log:
- Checksum/Hash:
- Timeline:

Failure/Exception:
Manual Steps:
Cleanup/Rollback:
Follow-up:
```

---

## 15. MVP Acceptance

Gate G(최종) 판정은 본 역할만으로 완료되지 않으며, Kubernetes/Application Integration, Delivery/Observability, Browser/E2E와 함께 Cross-role로 연결된다.

### 15.1 현재 본 역할 검증 상태

| 항목 | 현재 상태 |
| --- | --- |
| MariaDB/MaxScale Provider(Gate A) | PASS |
| Vault/Secret/TLS(Gate B) | PASS |
| NFS/Storage(Gate C) | PASS |
| MariaDB Backup Chain(Gate D) | PASS |
| MariaDB Recovery 절차(Gate E) | PASS(절차) |
| MariaDB DR-01 RTO/RPO(Gate E) | **PASS(완료, 이슈 #194)** |
| Redis Persistence(Gate F) | PASS |
| Redis DR-03 1차 Infra 레벨 검증(Gate F) | **PASS(격리 환경, 이슈 #115)** |
| Redis DR-03 운영 자동화(Gate F) | Not Tested / Deferred(BLOCKING: RBAC 결정) |
| etcd DR-02 스냅샷/전송(Gate G) | PASS(#113) |
| etcd DR-02 E2E 수동(Gate G) | PASS, RTO 38분44초(#156, 참고값) |
| etcd DR-02 E2E 자동화(Gate G) | PASS, RTO 52초(#176/PR#191, 대표값) |
| node_exporter | PASS |
| mysqld_exporter 수집 | PASS(수집), Alert 범위 밖 |
| maxscale_exporter | Deferred |
| cron orphan 코드 정리 | Not Tested |
| Application Data Consumer(Cross-role) | PASS(A-10 기준, 변경 시 재검증) |

### 15.2 최종 MVP Acceptance에 필요한 남은 검증

- Redis DR-03 운영 자동화 착수 여부를 Go/No-Go로 결정하고 근거 기록(BLOCKING: `platform/redis-0` 쓰기 반영 권한 결정)
- cron orphan 코드 레벨 정리(`state: absent`) 및 재검증
- maxscale_exporter는 2차 프로젝트 이관 여부를 최종 확정하고, 그 근거를 Acceptance 문서에 남김
- Application Data Consumer 연결의 최신 Revision 기준 재검증(DB/NFS/Backup/Recovery 자산 변경 시)

이미 PASS한 Gate A~E, F(Persistence + DR-03 1차 검증), G 항목을 다시 미완료로 표현하지 않으며, 아직 실행하지 않은 Redis DR-03 운영 자동화/maxscale_exporter/cron 정리 항목을 완료된 결과처럼 기록하지 않는다.

## 16. 문서 완료 기준

- 02 M-04/M-05 Traceability 유지.
- 09 Gate A~G Traceability와 정합.
- 11 실행절차 반복 없이 Test Contract로 연결.
- 각 Test의 Preconditions / Stimulus·Fault / Observation / PASS·FAIL / Evidence 명확.
- 정량 항목의 단위와 측정 출처 명확.
- 미실행/보류(Redis DR-03 운영 자동화, maxscale_exporter, cron 정리) 항목을 결과처럼 작성하지 않음.
- MariaDB DR-01·etcd DR-02는 완료된 실측값으로 명확히 표시하고, 근거 없이 재차 "미확정"으로 표현하지 않음.
- Redis DR-03의 "1차 Infra 레벨 검증 완료"와 "운영 자동화 완료"를 구분해서 표시.
- etcd DR-02 RTO 52초(대표값)의 측정 범위(K8s 상태 복구만)를 벗어난 해석 금지, 38분44초(#156)는 참고값으로만 병기.
- Cross-role Owner 경계 명확(Application Data Consumer, M-01/M-02/M-03).
- Secret/Vault/Token Evidence 금지.
- 실패 Run 보존, 재실행은 새 Run ID.
- 실제 Repository/Runtime과 재대조 후 추가 필수 보완사항이 없을 때 종료.
