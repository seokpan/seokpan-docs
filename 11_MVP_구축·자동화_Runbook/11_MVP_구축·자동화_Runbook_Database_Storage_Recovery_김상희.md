# MVP 구축·자동화 Runbook
## Database / Storage / Recovery

## 1. 목적과 사용 범위

이 문서는 「石나가는 판단」 1차 프로젝트의 Database / Storage / Recovery 영역(MariaDB/MaxScale, NFS, MariaDB Backup/Recovery, Redis Persistence, etcd DR, DB/NFS Observability Exporter)을 실제로 구축·적용·확인·재실행·복구·Rollback하기 위한 실행 Runbook이다.

09 문서가 책임·Provider/Consumer·Integration Gate(A~G)를 정의한다면, 본 문서는 각 Gate를 실제 작업으로 통과하는 절차를 소유한다.

```text
09 = What / Why / Responsibility / Gate
11 = Pre-check / Apply / Verify / Re-run / Recovery / Rollback
12 = Test Case / Measurement / PASS·FAIL / Evidence
```

본 문서는 현재 구현·실측 상태를 기준으로 작성한다. MariaDB Backup Chain, MariaDB DR-01(RTO/RPO), etcd Snapshot/Restore(DR-02), Redis DR-03(MariaDB 기준 재구성)은 실측 완료 항목이며, Redis DR-03의 운영 반영 자동화와 mysqld_exporter Alert Rule/maxscale_exporter 연동은 아직 완료되지 않은 항목으로 별도 표시한다.

직접 기준:

- [`09_MVP_실행·통합_실시설계_Database_Storage_Recovery_김상희.md`](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Database_Storage_Recovery_김상희.md)
- [`10_GitHub_협업_및_Repository_운영.md`](../10_GitHub_협업_및_Repository_운영/10_GitHub_협업_및_Repository_운영.md)
- [`PROJECT_CHANGES.md`](../PROJECT_CHANGES.md)
- [`MVP_IMPLEMENTATION_BASELINE.md`](../MVP_IMPLEMENTATION_BASELINE.md)
- `seokpan-infra` 현재 `main`, Issue #113/#115/#143/#156/#176/#194/#199/#211, PR #104/#117/#125/#130/#131/#133/#137/#145/#152/#168/#171/#191/#197/#200/#201
- `F_redis_recovery_contract.md`, `E_infra_snapshot.md`(9-9~9-11절), `H_progress_checklist.md`(F-9/F-10/I절)
- 실제 Runtime/Recovery Evidence

현재 상태는 계속 변할 수 있으므로 실행 직전에는 반드시 `seokpan-infra`의 최신 `main`과 열린 Issue/PR을 다시 확인한다.

---

## 2. 실행 원칙

### 2.1 Source of Truth와 Working Directory

```text
Ansible Role / Playbook / Inventory  → seokpan-infra (ansible/roles, ansible/playbooks, ansible/inventory/hosts.yml)
Vault Credential                     → seokpan-infra (group_vars/all/vault.yml)
Kubernetes StorageClass / PVC        → seokpan-gitops
Project Runbook / Evidence           → seokpan-docs
```

Ansible 실행 명령은 `seokpan-infra` Repository Root, Ansible Controller(`ans`, 192.168.54.70)에서 수행한다.

### 2.2 Master/Replica 동적 확인 우선

MaxScale `auto_failover=true`로 Master/Slave 역할이 언제든 바뀔 수 있다. **모든 DB 쓰기·백업·복구 작업 전에 반드시 `maxctrl list servers`로 현재 Master를 재확인**한다. 과거 세션 기록의 Master/Slave 배치를 그대로 신뢰하지 않는다.

### 2.3 상태 판정

다음을 같은 의미로 취급하지 않는다.

```text
Implemented ≠ Merged ≠ Running ≠ Validated
Backup 파일 존재 ≠ Backup Chain 정상 ≠ Restore 가능 ≠ Recovery 완료
Database Ready ≠ Backup Ready ≠ Recovery Ready ≠ Application Consumer Ready
```

### 2.4 Secret 비노출

Password, 전체 DB URL, Vault 평문 값, Token은 콘솔·Issue·PR·문서에 출력하거나 기록하지 않는다. 필요한 경우 Secret의 존재와 Key 이름만 확인한다. `--check --diff` 사용 시 `diff: false`가 적용된 태스크인지 먼저 확인한다.

### 2.5 원본 보존 원칙

백업 시 원본은 보존·단순 저장하고, 복구 시에만 필요한 만큼 가공한다. 실패한 Migration/Backup/Recovery Job을 재사용하지 않고 새로 생성한다.

### 2.6 중단 우선

아래 조건에서는 다음 단계로 진행하지 않는다.

- 현재 Master 서버 미확인(`maxctrl list servers` 미실행)
- 유효한 Backup Chain 부재
- NFS 공유 마운트(`/mnt/nfs-db-backup`)와 로컬 스테이징(`/srv/nfs/db-backup`) 상태값 불일치 의심
- Active Recovery/Migration Job 존재
- 백업 대상 디렉터리에 예기치 않은 `immutable`(`chattr +i`) 속성 존재
- etcd Snapshot Hash 불일치
- Vault ↔ Kubernetes Secret ↔ 실제 DB 계정 값 불일치 의심

---

## 3. 현재 실행 기준 상태

| 영역 | 상태 | 근거 |
| --- | --- | --- |
| MariaDB 2대 + MaxScale 구성 | Validated | 09 Gate A |
| DB 계정 체계(6개 계정, Vault 개별 변수) | Validated | PR #104 merge |
| MariaDB TLS(TLS 1.3, SAN 포함) | Validated | PR #137 merge, 09 Gate B |
| NFS Server / Kubernetes NFS Provisioner | Validated | 09 Gate C |
| MariaDB Backup Full+Incremental Chain | Validated | PR #117/#125 merge |
| Backup 공유 상태(NFS) / Shared Lock | Validated | Infra #166 PR #168 merge |
| Backup 동시성 immutable 방지 Guard | Validated | Infra #167 PR #171 merge |
| MariaDB Recovery 절차(`mariadb_dr_recovery.yml`) | Validated | Infra #119 PR #133 merge |
| MariaDB DR-01 실측(RTO/RPO) | **Validated(완료, 2026-09-18 close)** | Infra #194 — Isolated RTO 27.376초 / Production RPO 3건·RTO 1분29초(단일)·4분(이중화). PR #197로 `replication_setup.yml:117` Gate 태스크의 `when` 누락 버그 수정 |
| Redis Write/Read/PVC Persistence | Validated | 09 Gate F, seokpan-gitops#7 |
| Redis DR-03(PVC 손상 시 MariaDB 기준 재구성) | **Validated(1차 Infra 레벨, 격리 환경, 2026-09-18)**, 운영 자동화는 미착수 | Infra #115(open) — §8 참고 |
| etcd Snapshot 생성/무결성/NFS 전송 | Validated(축소 스코프) | Infra #113(2026-09-08 스코프 축소 후 completed) |
| etcd DR-02 E2E(격리 3-member Restore, 수동) | Validated, RTO 38분44초(트러블슈팅 포함) | Infra #156(2026-09-11) |
| etcd DR-02 E2E(자동화, Safety Guard/Restore Decision Gate) | Validated, RTO 52초 | Infra #176, PR #191(2026-09-16 merged) |
| node_exporter(4대: mariadb-01/02, maxscale-01, nfs) | Validated | Infra #199, idempotency 2회차 changed=0 |
| mysqld_exporter 배포·Prometheus 수집 | Validated(수집까지), Alert Rule 미착수 | Infra #211, 2026-09-21 스코프 확정 |
| maxscale_exporter | Deferred(2차 프로젝트) | 공식 exporter 부재(MXS-3022 Won't Do) |
| 레거시 cron orphan 코드 정리(`state: absent`) | Not Done | 수동 제거만 완료, 코드 레벨 정리 남음 |

이 표는 재실행·장애 대응 시 사용할 Current State 기준점이며, 12 문서의 Test Case별 PASS/FAIL Evidence를 대신하지 않는다.

> **Redis DR-03 관련 정정**: 이슈 #115가 "미완료"인 이유는 검증 자체가 안 끝나서가 아니라, 이번 검증에서 확정한 절차(§8)를 실제 운영 `platform/redis-0`에 자동 반영하기 위한 Ansible 자동화(`redis-dr-recovery-automation-handoff.md`로 인계)와 그 전제조건인 운영 Pod 쓰기 권한 결정이 아직 남아 있기 때문이다. 장애 관찰용 권한(`pods:delete`, `pods/log:get`)은 이미 `platform/ksh` SA에 부여 완료(seokpan-gitops#47/#48)했고, `pods/exec`는 부여하지 않기로 확정했다.

---

## 4. 공통 Pre-check

### 4.1 Repository 상태

```bash
cd <seokpan-infra-repo-root>
git status
git branch --show-current
git fetch --prune
git log --oneline --decorate -n 5
```

실제 변경은 별도 Branch에서 수행한다(예: `feature/194-mariadb-dr-measurement`, `infra/211_mysqld-maxscale-exporter`).

### 4.2 현재 Master/Replica 확인 (모든 작업 공통 선행)

```bash
maxctrl list servers
```

최소 확인:

```text
Master 서버 1대
Slave 서버 1대
Server State = Running
```

이 값은 세션마다 달라질 수 있으므로 이전 기록(예: 특정 날짜의 Master=mariadb-01)을 근거로 사용하지 않는다.

### 4.3 MariaDB / MaxScale Health

```bash
# 각 MariaDB 서버
mysqladmin -h <host> -u <monitor-user> -p status

# MaxScale
maxctrl list services
maxctrl list servers
```

### 4.4 NFS 마운트 상태

```bash
# NFS 서버(192.168.54.50)
exportfs -v

# 마운트하는 각 호스트
mount | grep nfs
```

**경로 혼동 주의(이슈 #194 실측 중 실제로 발견된 문서 오류 정정 사항)**:

```text
/srv/nfs/db-backup            → 각 호스트의 "로컬" 스테이징 디렉터리 (이름과 달리 NFS 공유 경로 아님)
/mnt/nfs-db-backup/.state/    → 실제 NFS 공유 상태 authority 경로 (.backup_chain_state.json이 있는 곳)
```

과거 `/mnt/nfs-db-backup` **루트**에 상태 파일이 있다고 기록된 적이 있으나, 실제 스크립트(`seokpan_mariadb_backup_chain.sh`)의 `STATE_FILE` 변수 기준으로는 `.state/` 서브디렉터리 하위다. 앞으로 상태 파일을 참조할 때는 반드시 아래 경로를 사용한다.

```bash
df -h /srv/nfs/db-backup            # 로컬 스테이징(호스트별 개별 확인)
df -h /mnt/nfs-db-backup            # NFS 공유 마운트
cat /mnt/nfs-db-backup/.state/.backup_chain_state.json
```

### 4.5 Immutable 속성 Fail-Fast Guard 확인

```bash
lsattr /srv/nfs/db-backup
```

`----i---------` 등 immutable 표시가 있으면 원인 파악 전까지 Play를 진행하지 않는다. 자동 해제(`chattr -i`)는 하지 않고 원인 규명을 우선한다.

### 4.6 Vault / Kubernetes Secret 정합성

```bash
ansible-vault view group_vars/all/vault.yml --vault-password-file <path>   # 값은 화면에만 표시, 기록 금지
kubectl -n application get secret backend-db-runtime -o go-template='{{range $k,$v := .data}}{{println $k}}{{end}}'
```

Key 이름만 대조하며, 여러 PR에 걸쳐 vault 값이 바뀐 이력(오염→복구 등)이 있으므로 항상 최신 merge 커밋 기준 값을 재확인한다.

### 4.7 etcd 클러스터 상태(DR-02 관련 작업 전)

```bash
etcdctl endpoint status --cluster -w table
etcdctl endpoint health --cluster
```

최소 확인: 3-member 모두 응답, Leader 1개, DB Size 이상치 없음.

---

## 5. NFS / Kubernetes Storage Runbook (Gate C)

### 5.1 NFS Export 확인

```bash
showmount -e 192.168.54.50
```

기대 Export:

```text
/srv/nfs/k8s
/srv/nfs/db-backup
```

### 5.2 Kubernetes StorageClass / Provisioner 확인

```bash
kubectl get storageclass
kubectl -n <provisioner-namespace> get pods -l app=nfs-subdir-external-provisioner
```

### 5.3 PVC Persistence 재검증

```bash
kubectl get pvc -A
kubectl delete pod <target-pod> -n <ns>   # 재생성 유도
kubectl exec -n <ns> <new-pod> -- cat <persisted-file-path>
```

Pod 재생성 후에도 이전 데이터가 유지되는지 직접 확인한다.

### 5.4 실패 시

NFS 서버 자체 장애 시:

```text
NFS 서버 상태 확인
→ exportfs 재확인
→ 마운트 클라이언트 재마운트
→ Kubernetes Pod Restart 필요 여부 판단(PVC bound 유지 시 자동 복구되는지 확인)
```

---

## 6. MariaDB Backup Runbook (Gate D)

### 6.1 선행조건

- 4.2의 현재 Master 확인 완료(Backup은 **Replica 기준**으로 수행)
- NFS 공유 마운트 정상
- Immutable 속성 없음(4.5)
- Active Backup Job 없음

### 6.2 Backup Chain 상태 확인

```bash
cat /mnt/nfs-db-backup/.state/.backup_chain_state.json
```

확인 항목: `chain_origin`, 최근 Full/Incremental 시각, 유효 Chain 여부.

### 6.3 Full Backup 실행 (수요일 14:00 스케줄, 수동 실행 시 동일 절차)

```bash
ansible-playbook -i inventory/hosts.yml playbooks/mariadb_backup_chain.yml \
  -e "backup_type=full" \
  --limit <current-replica-host>
```

### 6.4 Incremental Backup 실행 (월/화/목/금 스케줄)

```bash
ansible-playbook -i inventory/hosts.yml playbooks/mariadb_backup_chain.yml \
  -e "backup_type=incremental" \
  --limit <current-replica-host>
```

`backup_chain.sh.j2` 내부 로직이 이번 주기 유효 체인 여부를 자동 판정하며, 유효 Chain이 없으면 Full로 자동 승격한다(이슈 #129 로직).

### 6.5 실행 중 공유 Lock 확인

```bash
# 실행 중인 세션에서
flock -n /mnt/nfs-db-backup/.state/.backup_chain.lock -c 'echo would-block-if-locked'
```

정상적으로는 백업 실행 중 다른 호스트의 동시 백업 시도가 Lock 대기(최대 `backup_transfer_lock_wait_seconds=60`초)로 직렬화된다.

### 6.6 완료 후 확인

```bash
cat /mnt/nfs-db-backup/.state/.backup_chain_state.json
sha256sum /srv/nfs/db-backup/<latest-backup-dir>/*.qp
```

확인 항목:

```text
chain_origin 갱신
SHA-256 무결성
GTID 기록
7일 Retention 정책상 오래된 백업 정리 여부
```

### 6.7 레거시 cron 관련 주의

`ansible.builtin.cron`은 `name:` 마커로만 항목을 관리하므로, 스케줄 변경(이슈 #144) 이후 구 이름(`seokpan_mariadb_backup.sh`) 항목이 orphan으로 남을 수 있다. 현재 양쪽 서버 모두 수동 제거는 완료했으나 코드 레벨(`state: absent` 태스크) 정리는 아직 남아 있으므로, 코드만 신뢰하지 말고 실행 전 `crontab -l`로 직접 확인한다.

```bash
crontab -l -u <backup-svc-user>
```

### 6.8 실패 시

```text
Backup 실패
→ mkdir EPERM 계열 → immutable 속성(4.5) 재확인
→ NFS 마운트 단절 → 마운트 재확인(4.4)
→ "이번 주 유효 체인 없음" 오판 → .backup_chain_state.json 원본(공유 경로) 재확인
→ 원인 수정 후 재실행(동일 Job 재사용 금지, 상태 파일 기준으로 자연스럽게 새 Chain 판단되도록 함)
```

---

## 7. MariaDB Recovery Runbook (Gate E, DR-01)

### 7.1 선행조건

- 유효한 Backup Chain 존재(6.6 확인 완료)
- Recovery 대상이 `backup_restore_mode: isolated`(격리 인스턴스)인지 `production`(실제 서비스 datadir)인지 명시적으로 구분 — 이슈 #119/PR #133에서 두 모드를 분기 구현
- Migration/Runtime Secret과 별개로 Recovery 전용 절차 진행

> **정정**: 이전 버전 문서에서 이 항목의 근거로 인용했던 "이슈 #176"은 실제로는 etcd DR-02 자동화 전용 이슈(Safety Guard/Restore Decision Gate)이며 MariaDB Recovery와는 무관하다. MariaDB 쪽의 잘못된 환경 실행 방지는 §9.1이 아니라 위와 같이 `backup_restore_mode` 분기와 Split-brain 방지 안전장치(§7.6 인접 설계, 이슈 #119)로 구현되어 있다.

### 7.2 실행 순서 (09 문서 3.11 흐름 기준)

```text
Backup Chain 확인
→ Full Backup Prepare
→ Incremental Backup 순차 Prepare
→ 복구 데이터 구성
→ MariaDB 기동
→ Replication 재구성
→ MaxScale 상태 확인
→ 서비스 접근 확인
→ 데이터 정합성 검증
```

### 7.3 실행 명령

```bash
ansible-playbook -i inventory/hosts.yml playbooks/mariadb_dr_recovery.yml \
  -e "backup_restore_role=<master|replica>" \
  --limit <recovery-target-host>
```

`backup_restore_role`은 실행 대상이 Master 경로인지 Replica 경로인지 명시적으로 지정한다. PR #197 이전에는 `roles/backup_transfer/tasks/replication_setup.yml:117`의 "복제 정상 기동 Gate" 태스크에 `when: backup_restore_role == 'replica'` 조건이 누락되어 있어, `master` 경로 실행 시 register되지 않은 `dr_slave_status.query_result`를 참조하다 fatal 처리되는 문제가 있었다(같은 파일의 `CHANGE MASTER TO`/`START SLAVE`/`SHOW SLAVE STATUS` 3개 태스크는 이미 정상적으로 조건이 걸려 있었고, 이 Gate 태스크만 예외적으로 누락되어 있었음). 실제 데이터 영향은 없었으나(필요한 작업은 이미 정상 스킵된 시점), 최신 `main` 기준 Playbook인지 먼저 확인한다.

### 7.4 실행 후 확인

```bash
mysql -h <recovered-host> -e "SHOW SLAVE STATUS\G"
maxctrl list servers
```

최소 확인 대상:

```text
복구된 데이터 건수
데이터 무결성 / 외래키 관계
GTID 상태
Replication 상태(Slave_IO_Running / Slave_SQL_Running = Yes)
Application 계정(identity_svc, game_svc) 접근 가능 여부
MaxScale 상태
서비스 접근 가능 여부
```

### 7.5 RTO/RPO 측정 — 이슈 #194 확정 실측값(2026-09-16, 재실행 시 이 값을 기준선으로 사용)

Recovery는 시나리오에 따라 RTO/RPO 정의가 달라지므로, 재실행 시에도 아래 두 시나리오를 구분해서 측정·기록한다.

**Isolated 모드(Full-only 체인, 격리 인스턴스)**

```text
mariadb_restore_chain.yml, mariadb-02 대상, chain_20260916(Full 1개, Incremental 0개)
RTO = 27.376초
Count/CHECKSUM/FK Gate 7개 테이블·7개 관계 전부 PASS
```

**Production 모드(양쪽 동시 유실 최악 시나리오)**

```text
절차: 더미데이터 A 삽입(Master) → 복제 반영 확인 → Incremental 백업 수동 트리거(A까지 포함)
    → 더미데이터 B 삽입(백업 미포함, 유실 대상) → mariadb-01/02 양쪽 동시 정지
    → mariadb_dr_recovery.yml 4단계(restore→replication→maxscale_verify→checklist)를
      mariadb-01(role=master) → mariadb-02(role=replica) 순으로 각각 완주

RPO = 3건 손실(member 10건→7건, 더미데이터 B 전량 — Incremental 이후 미백업 데이터가 설계대로 정확히 손실)
RTO(단일 노드 서비스 재개) = 1분 29초
RTO(양쪽 이중화 완전 정상화) = 4분 0초
```

Split-brain 방지 안전장치(비정상 노드에 read_only 강제 배포)의 최초 실측 검증도 이때 완료됐다 — mariadb-02가 정식 Replica로 재편입되는 시점에 안전장치가 자동 해제됨을 확인.

> **이 값들은 이미 확정·종결된 최종 결과다**(Infra #194, 2026-09-18 completed close). 09/12 문서에도 이 값을 그대로 반영한다. 다만 이 수치는 해당 실측 시점의 Backup Chain 구성(Full 1개 + 더미 Incremental)과 서버 사양에 종속된 값이므로, Chain 구성이나 서버 환경이 달라지면 새 Run으로 재측정하고 이전 값을 덮어쓰지 않는다(12 문서 §4.2 실패/과거 Run 보존 원칙과 동일하게 이전 Run도 이력으로 남긴다).

### 7.6 Master/Replica 양쪽 재검증

PR #197 반영 후에는 위 Production 모드 실행 자체가 Master 경로, Replica 경로를 순서대로 각각 통과하므로 별도 재검증이 필요하지 않다. 코드만 변경되고 재측정이 필요 없는 경우에는 `--check`로 사전 확인 후 판단한다.

```bash
ansible-playbook -i inventory/hosts.yml playbooks/mariadb_dr_recovery.yml -e "backup_restore_role=master" --limit <host> --check
ansible-playbook -i inventory/hosts.yml playbooks/mariadb_dr_recovery.yml -e "backup_restore_role=replica" --limit <host> --check
```

### 7.7 실패 시

```text
Recovery 실패
→ Job/Log 확인
→ DB 상태 확인
→ Replication 확인
→ 원인 수정
→ 새 승인/새 실행(기존 실패 실행 결과 재사용 금지)
```

Schema/데이터 손상이 의심되면 Runtime 서비스로의 전환을 중단하고 별도 격리 환경에서 재검증한다.

---

## 8. Redis DR-03 Runbook (Gate F) — PVC 손상 시 MariaDB 기준 재구성

### 8.1 설계 원칙 (`F_redis_recovery_contract.md` 기준)

Redis DR-03은 "Redis 데이터를 별도로 백업해뒀다가 복원"하는 방식이 **아니다**. 핵심 원칙은 다음과 같다.

```text
MariaDB의 확정 기록(move, game_result, rating_history)이 항상 "진실"이다.
Redis가 담당하는 것은 Room/현재 투표/Ready 상태 같은 "지금 이 순간"의 공유 런타임 상태뿐이다.
Redis 복구 후 상태가 MariaDB와 어긋나면 MariaDB 기준으로 Redis를 맞춘다(반대 방향 아님).
```

시나리오별 기대 결과:

```text
Redis Pod만 재시작(PVC 그대로)        → AOF Replay로 완전 복구
Redis Pod가 다른 노드로 이동           → 완전 복구(NFS가 원본 저장소이므로 노드 장애 ≠ 데이터 유실)
PVC 자체 손상/유실                    → "진행 중" 상태(현재 투표, 턴 진행)는 유실 인정, MariaDB 기준 재구성
AOF 마지막 fsync 이후 구간             → 최대 1초 분량 유실 가능(everysec 정책의 표준적 한계)
```

### 8.2 기본 Persistence 재검증(Pod 재시작 시나리오)

```bash
kubectl -n platform get statefulset,pod,pvc redis-0 2>/dev/null
kubectl -n platform exec redis-0 -- redis-cli SET dr_check_key "$(date +%s)"
kubectl -n platform delete pod redis-0
kubectl -n platform exec redis-0 -- redis-cli GET dr_check_key
```

값이 유지되면 AOF/PVC 기반 Persistence PASS(이미 seokpan-gitops#7에서 검증 완료된 범위).

### 8.3 PVC 손상 시 재구성 절차 (격리 환경 기준, 1차 Infra 레벨 검증 완료 — 이슈 #115, 2026-09-18)

**운영 `platform/redis-0`에는 아직 이 절차를 자동으로 실행하지 않는다.** 실제 검증은 동일 스펙(redis:8.10.1, appendonly yes/everysec, nfs-k8s PVC)의 격리 StatefulSet(`storage-infra/redis-dr03-test`)에서 수행했다. 운영 Redis에 대한 자동화는 §8.6의 선행조건이 해결된 이후 진행한다.

재구성 규칙:

```text
1. MariaDB의 마지막 확정 Move를 기준으로 삼는다
   → game_id로 move 테이블을 move_no 순으로 조회해 보드를 재구성
2. 다음 턴 차례는 마지막 Move의 반대 팀으로 판단
3. Pass 이력은 MariaDB에 남지 않으므로 재구성된 turn_no는 근사치임을 인정
   → "게임 재개 시 이번 턴부터 다시 투표"로 처리
4. 아직 MariaDB에 확정되지 않은 진행 중 투표 상황은 복구 대상이 아님(유실돼도 정상 동작)
```

실제 Redis Key 구조(운영 데이터 실측 확인, `stone:v1:room:{room_id}:` prefix):

```text
game     (STRING/JSON) → 재구성 대상, MariaDB move 기준
board    (HASH, 좌표→team) → 재구성 대상
requests / request-expiries (TTL 20h) → 재구성 대상 아님(4번 규칙의 "복구 불가능한 활성 상태")
```

좌표 변환: `좌표문자 = chr(ord('A') + pos_x)`, `좌표숫자 = pos_y + 1`

### 8.4 MariaDB ↔ Redis 상태 일치 판정

```text
일치           : Redis의 "확정된 마지막 착수" = MariaDB 최신 Move(game_id+move_no 기준)
불일치(Redis 앞섬)  : Redis에는 있는데 MariaDB에는 없음 → 무효 처리("일어난 적 없는 일")
불일치(Redis 뒤처짐) : MariaDB에는 있는데 Redis 미반영 → MariaDB 기준으로 강제 재동기화
```

### 8.5 검증된 절차 (격리 환경 재현 순서)

```text
격리 StatefulSet(redis-dr03-test) 배포
→ MariaDB synthetic 데이터 적재
→ 정상 재기동 AOF Replay 복구 확인
→ AOF(base.rdb) 직접 손상 → CrashLoopBackOff 재현("Wrong RDB checksum ... RDB CRC error")
→ MariaDB move_no 기준 SQL 수동 재구성 → 사전 계산값과 완전 일치 확인
→ 손상 파일 격리(rename 보존) + 재구성값 반영으로 서비스 재개
→ Redis Ahead(무효화)/Behind(재동기화)/Exact 3케이스 판정 검증
→ Data Loss/Duplicate/Stale 3종 무결성 검증(전부 없음 확인)
```

작업 중 HAProxy idle timeout(60초) 및 MaxScale readwritesplit 쓰기-직후-읽기 불일치를 다시 겪었으나(`G_backend_db_connection_guide.md`에 이미 문서화된 기지 현상), 신규 결함은 아니다.

### 8.6 운영 반영 전 남은 선행조건 (BLOCKING)

```text
1차 수동 절차 검증 완료(§8.3~8.5)
→ Ansible 자동화 착수(redis-dr-recovery-automation-handoff.md로 Claude Code 인계)
→ 운영 platform/redis-0에 대한 재구성값 "쓰기" 반영 권한 결정  ← 아직 미확정, BLOCKING
→ 운영 Redis 자동 복구 절차 확정
```

장애 관찰/주입용 권한은 이미 부여됐다(`platform/ksh` SA, `platform/redis-0` 대상 `pods:delete`+`pods/log:get`, seokpan-gitops#47/#48). `pods/exec`는 부여하지 않기로 확정했으므로, 재구성값을 운영 Redis에 실제로 써넣는 방식(예: 별도 서비스 계정 경유, Job 방식 등)은 별도로 설계·결정해야 한다. 이 결정 전까지 이슈 #115는 open 상태를 유지한다.

---

## 9. etcd Snapshot / Restore Runbook (Gate G, DR-02)

### 9.0 이슈 이력 (재실행 전 반드시 구분해서 참조)

```text
#113 (2026-09-08 스코프 축소, completed)
  → Snapshot 생성 + SHA-256/무결성 검증 + NFS 전용 경로(/srv/nfs/etcd-dr) 전송까지만

#156 (2026-09-11, closed) — 1차 수동 E2E
  → 별도 물리PC 격리망(loadgen/loadgen2/loadgen3, VMnet 192.168.55.0/24)에서
    수동으로 3-member Restore → K8s API 기동 → Object 검증 → RTO 측정
  → RTO = 38분 44초(트러블슈팅 대응 시간 포함), Namespace 12/Deployment 21/Secret 37 diff 0

#176 + PR #191 (2026-09-14~16 merged) — 자동화
  → Safety Guard → Restore Decision Gate → Final Restore Gate 3단 자동화 파이프라인 구현
  → bridged loadgen 환경(10.1.93.95~97/24)에서 E2E 자동 실행
  → RTO = 52초(Restore→Quorum 21초 / Quorum→API 27초 / API→Object 4초), Object diff 0
```

이 Runbook의 §9.6~9.7은 **자동화된(PR #191) 절차**를 기준으로 서술한다. 12 문서와 09 문서에서 "etcd DR-02 RTO"를 인용할 때는 이 자동화 결과(52초)를 대표값으로 사용하고, #156의 38분44초는 "자동화 이전 수동 절차 참고값"으로만 병기한다.

### 9.1 선행조건

- 4.7의 etcd 클러스터 상태 확인 완료
- `/opt/seokpan/etcd-tools/current/`에 배포된 도구의 SHA-256 일치 확인(PR #152)
- cp-03 등 재부팅 이력이 있는 노드는 NetworkManager ens160 초기화 지연 관련 CrashLoopBackOff 이력이 데이터 손상이 아님을 확인

### 9.2 도구 검증

```bash
sha256sum /opt/seokpan/etcd-tools/current/etcdctl /opt/seokpan/etcd-tools/current/etcdutl
```

배포 시점의 공식 SHA-256 값과 대조한다.

### 9.3 Snapshot 생성

```bash
ETCDCTL_API=3 etcdctl snapshot save /tmp/snapshot-$(date +%Y%m%d_%H%M%S).db \
  --endpoints=https://<etcd-endpoint>:2379 \
  --cacert=<ca> --cert=<cert> --key=<key>
```

`etcdctl snapshot save`를 사용한다(`etcdutl snapshot save` 아님 — PR #152에서 정정 확인된 사항).

### 9.4 Snapshot 검증

```bash
ETCDCTL_API=3 etcdutl snapshot status /tmp/snapshot-<ts>.db -w table
sha256sum /tmp/snapshot-<ts>.db
```

확인 항목: HASH, REVISION, TOTAL KEYS, DB Size를 4.7에서 기록한 Raft Index/Term 시점 값과 대조한다.

### 9.5 NFS 전송

```bash
mkdir -p /srv/nfs/etcd-dr   # 최초 1회, nfs-utils / root_squash 설정 확인 후
rsync -av /tmp/snapshot-<ts>.db <nfs-mount>/etcd-dr/
```

`nfs-utils` 미설치나 `root_squash` 설정 충돌로 rsync가 실패한 이력이 있으므로, 전송 전 대상 마운트가 쓰기 가능한지 별도 확인한다.

```bash
touch <nfs-mount>/etcd-dr/.write-test && rm <nfs-mount>/etcd-dr/.write-test
```

### 9.6 격리 환경 Restore

```bash
ETCDCTL_API=3 etcdutl snapshot restore <nfs-mount>/etcd-dr/snapshot-<ts>.db \
  --name <isolated-member-name> \
  --initial-cluster <isolated-cluster-config> \
  --initial-advertise-peer-urls <peer-url> \
  --data-dir /var/lib/etcd-dr-restore
```

격리된 3-member 구성으로 Restore 후 아래를 확인한다.

```bash
etcdctl endpoint status --cluster -w table   # 격리 클러스터 대상
etcdctl endpoint health --cluster
```

최소 확인:

```text
etcd Leader / Raft 상태
Snapshot Hash / Revision 원본과 일치
3-member Quorum 성립
kube-apiserver 인증(격리 환경에 임시 apiserver 구성 시)
Namespace / Deployment / Secret Object 수 원본과 비교
```

### 9.7 RTO 측정

Restore 시작 시각 → Quorum 성립 및 kube-apiserver Object 비교 완료 시각까지를 측정한다. 최종 E2E 측정값은 **52초**(Infra PR #191)이며, 이는 etcd Snapshot 기반 Kubernetes 상태 복구 시간만을 의미하고 전체 클러스터 재구축·Worker 재가입 시간은 포함하지 않는다. 재측정 시에도 이 범위 정의를 그대로 유지한다.

### 9.8 스코프 경계 재확인

DR-02 스코프는 "K8s 서버는 살아있고 클러스터 내부 상태 데이터(etcd)만 손상된 경우"로 한정한다(2026-09-14 확정). "K8s 서버 자체 손상 시 신규 서버로 전체 이관 복구"(PKI 백업, 실제 CA 재구성 포함, 이슈 #178)는 본 프로젝트에서 구현하지 않으며 개념 정리만 유지한다. 이 경계를 벗어나는 요청이 들어오면 실행 전에 범위를 다시 확인한다.

### 9.9 실패 시

```text
Snapshot Hash 불일치 → 원본 재확인, 전송 경로 재검증(NFS 손상 가능성)
Quorum 미성립 → 격리 클러스터 구성 파일 재확인
kube-apiserver 인증 실패 → 격리 환경 인증서/설정 재확인
```

---

## 10. DB/NFS Observability Exporter Runbook

### 10.1 node_exporter (mariadb-01/02, maxscale-01, nfs)

```bash
ansible-playbook -i inventory/hosts.yml playbooks/node_exporter_linux.yml --limit db,maxscale,nfs
ansible-playbook -i inventory/hosts.yml playbooks/node_exporter_linux.yml --limit db,maxscale,nfs --check --diff
```

두 번째(재실행) 결과가 `changed=0`인지 확인해 idempotency를 재검증한다.

### 10.2 mysqld_exporter 0.20.0 (mariadb-01/02, PR #201)

```bash
ansible-playbook -i inventory/hosts.yml playbooks/mysqld_exporter.yml --limit db
```

`exporter_svc` 계정(Master 동적 판별로 Master에서만 DDL 수행, Replica는 복제로 전파)으로 동작하는지 확인한다.

```bash
mysql -h <mariadb-host> -u exporter_svc -p -e "SHOW GRANTS;"
curl -s http://<mariadb-host>:9104/metrics | grep -E 'mysql_up|slave_status'
```

**기지 버그(TS-043, 수정 완료)**: 배포 초기 `exporter_svc` GRANT에 `SLAVE MONITOR` 권한이 누락되어 `slave_status` collector가 `Access denied(1227)`를 발생시켰다. 재배포 시 이 권한이 GRANT에 포함되어 있는지 확인한다.

```bash
mysql -h <mariadb-host> -u exporter_svc -p -e "SHOW GRANTS;" | grep -i "SLAVE MONITOR"
```

정상 상태 기준값: `mysql_up 1`, `mysql_exporter_collector_success{collector="collect.slave_status"} 1`, `slave_io_running=1`, `slave_sql_running=1`.

Prometheus Scrape Target 등록과 Alert Rule(복제 지연, Slave 중단)은 2026-09-17 팀 결정으로 보류되었으므로 이번 문서 범위에서는 다루지 않는다. 방화벽(vrouter) `9100/tcp`(node_exporter) rich rule은 mariadb-01/02 대역(`192.168.51.0/24`, `192.168.52.0/24`)만 확인됐고 maxscale-01(`.53.0/24`)·nfs(`.54.0/24`) 대역 커버 여부는 네트워크 담당자 확인이 남아 있다. `9104/tcp`(mysqld_exporter) 방화벽 요청은 위 보류 결정에 따라 이번 라운드에서 진행하지 않는다.

### 10.3 maxscale_exporter (Deferred)

바이너리 배포는 시간 부족으로 재보류 확정(2026-09-18). `mariadb-corporation/maxscale_exporter`는 MariaDB JIRA MXS-3022가 Won't Do로 closed되어 존재하지 않으며, 커뮤니티 대안(`Vetal1977/maxctrl_exporter` 등)은 공식 릴리스 바이너리가 없어 `go build` 소스 빌드가 필요하다. 현재는 `maxscale_exporter`(basic 타입) REST API 계정 생성과 `playbooks/maxscale_exporter.yml`/`roles/maxscale_exporter` 골격까지만 존재하며, idempotency 및 재귀 검증(읽기 정상/쓰기 401 차단/서비스 상태 불변)은 통과했다. 실제 바이너리 배포·Prometheus 연동은 2차 프로젝트에서 Go 소스 빌드 방식(A안)으로 재착수한다.

### 10.4 확인해 둘 잔여 항목

기본 `admin`/`mariadb` REST API 계정이 아직 살아있는 상태가 발견되어 있다. 별도 이슈 등록 여부는 결정 대기 중이므로, 이 문서 기준으로는 보안 노출 여부만 확인하고 정리 작업은 별도 이슈로 분리한다.

```bash
curl -sk -u admin:<REDACTED> https://<maxscale-host>:8989/v1/servers  # 값 노출 없이 응답 코드만 확인
```

---

## 11. 장애 유형별 Recovery

| 증상 | 우선 확인 | 기본 조치 |
| --- | --- | --- |
| Backup mkdir EPERM | `lsattr` immutable 속성 | 원인 규명 후 수동 `chattr -i`, 재발 시 팀 공유 |
| "이번 주 유효 체인 없음" 오판 | 공유 경로(`/mnt/nfs-db-backup/.state/`)와 로컬 스테이징(`/srv/nfs/db-backup`) 상태값 불일치 | 공유 경로(`.state/.backup_chain_state.json`) 값을 기준으로 재확인, `state_set()` 원자성 확인 |
| Backup 이중 실행/Chain 오염 | `flock` 획득 로그, 동시 실행 여부 | Lock 대기시간(`lock_wait_seconds`) 설정값 확인 |
| NFS 마운트 단절 | `mount`/`df` 상태, NFSv4 콜백 지연 특성 | 재마운트, 폴링 주기(약 30초) 감안한 대기 |
| Recovery 시 Master 경로 fatal(register 안 된 변수 참조) | `replication_setup.yml:117` Gate 태스크의 `when` 조건 | PR #197 반영 여부 확인, 최신 `main` 사용 |
| Replication 재구성 실패 | GTID 갈림 여부 | auto_rejoin 실패 시 mariadb-backup 기반 수동 재구축 |
| DCL 복제 에러 | `slave_ddl_exec_mode=IDEMPOTENT` 미적용 대상(DROP USER 등) | `sql_slave_skip_counter=1` 적용 |
| etcd Snapshot Hash 불일치 | 전송 경로(NFS) 손상 여부 | 원본 재생성, 전송 재시도 |
| etcd Restore Quorum 미성립 | 격리 클러스터 구성 파일 | `initial-cluster` 등 파라미터 재확인 |
| Redis Pod 재시작만으로 복구 안 됨 | AOF 설정(`appendfsync`), PVC bound 상태 | PVC 유지 여부 확인, 정상이면 §8.2로 재시도 |
| Redis PVC 자체 손상/유실 | MariaDB 최신 Move와 Redis 상태 비교(§8.4) | §8.3 재구성 절차를 격리 환경에서 우선 재현 후 운영 반영 여부는 §8.6 선행조건 충족 후 판단 |
| mysqld_exporter 인증/수집 실패 | `exporter_svc` GRANT에 `SLAVE MONITOR` 포함 여부 | TS-043 패턴 재확인, 계정 권한 재확인 |
| Vault 값 참조 오류 | 여러 PR에 걸친 vault 값 변경 이력 | 최신 merge 커밋 기준 vault 값 재확인 |

장애 복구의 공식 측정값과 Evidence는 12에서 관리한다.

---

## 12. 재실행 / 멱등성 기준

### 12.1 Ansible 변경

`diff: false`로 비밀번호 평문 노출을 방지하고, `update_password: on_create`로 기존 계정 비밀번호를 보존하며, `append_privs: true`로 명시 외 기존 GRANT가 REVOKE되지 않도록 유지한다. 조회 태스크는 `check_mode: false`로 `--check` 모드에서도 실행되도록 유지해 사전 확인이 가능하게 한다.

### 12.2 Backup/Recovery

실패한 기존 Backup/Recovery Job을 재사용하지 않는다.

```text
실패
→ 원인 확인(immutable / NFS / Lock / GTID)
→ 상태 파일 재확인
→ 새 실행
```

### 12.3 cron 관리

`ansible.builtin.cron`은 `name:` 마커로만 관리되므로 이름 변경 시 구 항목이 orphan으로 남는다. 스케줄 변경 시 구 이름 항목에 대한 `state: absent` 태스크를 함께 추가하는 것을 원칙으로 한다(현재 미완 항목, §6.7).

### 12.4 etcd 재실행

Snapshot/Restore는 매번 새 파일명(`snapshot-YYYYMMDD_HHMMSS.db`)으로 생성하며, 기존 파일을 덮어써 재사용하지 않는다.

---

## 13. Rollback Matrix

| 변경 대상 | 기본 Rollback | 주의사항 |
| --- | --- | --- |
| MariaDB Backup 설정/스케줄 | Ansible 코드 Git Revert | cron 마커 이름 변경 시 orphan 재발 가능성 확인 |
| MariaDB Recovery 절차 | 격리 환경에서만 재시도, Runtime 서비스 직접 롤백 대상 아님 | 이슈 #176(잘못된 환경 실행 방지) 기준 준수 |
| Backup Chain 상태 파일 | 마지막 유효 상태로 수동 복원(공유 lock 하에서) | 파일시스템 간 `mv` 금지, 동일 디렉터리 내 원자적 교체 유지 |
| NFS 권한/immutable 속성 | 수동 `chattr -i` 후 원인 별도 추적 | 자동 해제 금지 |
| etcd Snapshot/Restore | 격리 환경 재구성, 실제 운영 클러스터 직접 적용 없음 | Restore 대상은 항상 격리 환경, 운영 클러스터 대체 아님 |
| Observability Exporter(node/mysqld) | Ansible Role Git Revert | idempotency 재검증(`changed=0`) |
| maxscale_exporter | 현재 코드(계정+골격)만 유지, 배포 롤백 대상 아님 | 2차 프로젝트 착수 전까지 변경 보류 |

---

## 14. 12 검증·측정 계획으로의 인계

11은 실행 방법을 제공하지만 다음은 12의 책임이다.

```text
정식 Test Case ID
RTO/RPO 확정값(DR-01/DR-02)의 Test Contract 반영
Backup Chain 무결성 회차별 결과
Redis DR-03 자동화 착수 여부와 Go/No-Go 근거
etcd DR-02 반복 측정값
Metric Query / Log / Run ID
최종 Evidence Link
```

11에서 실행한 절차는 12에서 재현 가능한 Test Case와 Evidence로 연결되어야 한다.

---

## 15. Runbook 완료 기준

- 실제 Repository 자산(Role/Playbook)과 실행 절차가 일치한다.
- Master/Replica 동적 확인을 모든 절차의 공통 선행조건으로 명시한다.
- Backup은 Replica 기준, Recovery는 격리 환경 기준임을 명확히 한다.
- NFS 공유 상태 파일의 두 경로(공유 마운트/로컬 스테이징)를 구분해 확인한다.
- Immutable Guard, 공유 Lock, cron 마커 등 알려진 함정을 절차에 반영한다.
- etcd DR-02 스코프 경계(내부 상태 손상 vs 서버 손상)를 절차에서 재확인한다.
- Redis DR-03의 "1차 Infra 레벨 검증 완료"와 "운영 자동화 완료"를 같은 의미로 쓰지 않는다.
- Observability Exporter의 보류 항목(Alert Rule, maxscale_exporter)을 완료로 표현하지 않는다.
- 실패 시 중단·복구·Rollback 경로가 있다.
- 09와 책임 중복이 없고 12의 Test/Measurement/Evidence를 침범하지 않는다.
- Current-State stale 내용은 `seokpan-infra` Issue/PR로 분리 추적한다.

실측이 진행되면서 RTO/RPO 확정값, Redis DR 범위, maxscale_exporter 착수 여부가 정해지면 본 Runbook의 Planned 절차를 실제 자산 기준으로 현행화한다.
