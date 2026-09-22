# 09_MVP_실행·통합_실시설계

## Database / Storage / Recovery

---

## 목차

1. [목적과 범위](#1-목적과-범위)
2. [선행 기준과 상태 판정](#2-선행-기준과-상태-판정)
3. [Database / Storage / Recovery Current State](#3-database--storage--recovery-current-state)
4. [Provider / Consumer Contract](#4-provider--consumer-contract)
5. [Integration 흐름](#5-integration-흐름)
6. [Integration Gate](#6-integration-gate)
7. [Gate 요약](#7-gate-요약)
8. [남은 Blocker와 Gap](#8-남은-blocker와-gap)
9. [Critical Path](#9-critical-path)
10. [완료 기준](#10-완료-기준)
11. [Traceability](#11-traceability)

---

# 1. 목적과 범위

## 1.1 목적

본 문서는 Kubernetes 기반 애플리케이션이 사용하는 **데이터 저장 계층(Database / Storage)** 과 장애 상황에서 데이터를 복구하기 위한 **Recovery / DR(Data Recovery / Disaster Recovery)** 구성을 실제 실행 및 검증 관점에서 정리한다.

단순히 MariaDB, Redis, NFS, etcd가 설치되어 있는지를 확인하는 것이 아니라 다음 흐름이 실제로 연결되는지를 검증하는 것을 목적으로 한다.

```text
Database 구성
    ↓
Application이 사용할 접속 정보와 계정 구성
    ↓
Kubernetes에서 Database 접근
    ↓
Database 데이터 저장
    ↓
Backup 생성 및 NFS 보호 저장
    ↓
장애 상황에서 Backup / Replication 기반 복구
    ↓
복구된 데이터의 정합성 검증
    ↓
Application이 복구된 Database를 다시 사용할 수 있는지 확인
```

또한 Kubernetes 자체의 핵심 데이터인 etcd와 애플리케이션에서 사용하는 Redis에 대해서도 각각의 저장 및 복구 상태를 구분하여 기록한다.

---

## 1.2 범위

| 영역               | 주요 구성                                                             | 검증 목적                     |
| ---------------- | ------------------------------------------------------------------- | ------------------------- |
| MariaDB          | mariadb-01 / mariadb-02                                              | 데이터베이스 운영 및 복제 상태 확인      |
| MaxScale         | maxscale-01                                                          | DB 접속 지점과 장애 전환 확인        |
| Database Schema  | `stone_game`                                                         | 애플리케이션 데이터 구조 확인          |
| Database Account | `identity_svc`, `game_svc`, `db_admin`, `backup_svc`, `repl_user`, `maxscale_monitor`, `exporter_svc` | 용도별 접근 권한 및 인증 확인 |
| TLS              | MariaDB TLS + Root CA                                                | Kubernetes → DB 암호화 연결 확인 |
| NFS              | `192.168.54.50`                                                      | 백업 및 Kubernetes 저장소 제공    |
| MariaDB Backup   | Full + Incremental                                                   | 백업 체인 생성 및 보존 확인          |
| MariaDB Recovery | `mariadb_dr_recovery.yml`                                            | 백업 기반 DB 복구 및 RTO/RPO 측정  |
| Redis            | `platform` namespace                                                 | 게임 상태 데이터 저장 및 영속성 확인     |
| Redis DR-03      | MariaDB 확정 기록 기준 재구성                                                | PVC 손상 시 복구 계약 검증         |
| etcd DR          | Kubernetes etcd Snapshot                                             | Kubernetes 상태 데이터 복구 확인   |

---

# 2. 선행 기준과 상태 판정

## 2.1 상태 표현 기준

본 문서에서는 다음 상태를 사용한다.

| 상태            | 의미                    |
| ------------- | --------------------- |
| `Defined`     | 설계 또는 구성 방법이 정의됨      |
| `Implemented` | 코드 또는 설정으로 구현됨        |
| `Merged`      | Git 저장소의 기준 브랜치에 반영됨  |
| `Running`     | 실제 환경에서 현재 동작 중       |
| `Validated`   | 실제 실행을 통해 정상 동작을 확인함  |
| `Partial`     | 일부 조건만 검증됨            |
| `In Progress` | 작업 또는 검증이 진행 중        |
| `Blocked`     | 다른 작업 또는 문제로 진행할 수 없음 |
| `Not Tested`  | 구현되어 있으나 실제 검증하지 않음   |
| `Deferred`    | MVP 범위에서 후순위로 보류      |

### 중요한 판정 원칙

```text
Implemented ≠ Merged ≠ Running ≠ Validated
```

Ansible Playbook이 존재한다는 사실만으로 실제 복구가 성공했다고 판단하지 않는다.

마찬가지로 다음 상태도 서로 분리한다.

```text
Database Ready
≠
Backup Ready
≠
Recovery Ready
≠
Application Consumer Ready
≠
Application Integration Validated
```

**추가 원칙(2026-09-18 재귀 검증 시 재확인)**: 하나의 DR 대상(Redis DR-03 등)에 대해서도 "격리 환경에서 절차 자체가 검증됨"과 "운영 환경에 그 절차가 자동으로 반영됨"은 서로 다른 판정이다. 전자가 `Validated`라고 해서 후자까지 `Validated`로 승격하지 않는다.

---

## 2.2 Recovery 관련 판정 기준

MariaDB의 경우 다음을 별도로 확인한다.

```text
Backup 생성 가능
    ↓
Backup Chain 정상
    ↓
Backup 보호 저장
    ↓
Backup Restore 가능
    ↓
Database 정합성 확인
    ↓
Replication 재구성
    ↓
MaxScale 상태 확인
    ↓
서비스 접근 가능
```

따라서 백업 파일이 존재한다는 것만으로 **MariaDB DR 완료**로 판단하지 않는다.

---

## 2.3 RTO / RPO 판정 기준

### RTO

**RTO(Recovery Time Objective)** 는 장애가 발생한 이후 서비스를 다시 사용할 수 있는 상태까지 복구하는 데 걸린 시간을 의미한다.

본 프로젝트에서는 실제 복구 절차를 실행하고 단계별 소요시간을 기록하여 최종 RTO를 측정한다.

### RPO

**RPO(Recovery Point Objective)** 는 장애가 발생했을 때 복구할 수 있는 데이터의 시점과 장애 발생 시점 사이의 차이를 의미한다.

본 프로젝트에서는 마지막으로 확보된 백업과 복구 대상 데이터의 시점을 비교하여 실제 RPO를 측정한다.

RTO/RPO는 시나리오에 따라 다른 값을 갖는다(예: MariaDB의 Isolated 복구와 Production 양쪽 유실 복구는 서로 다른 RTO/RPO를 가지며, 하나로 통합해 표현하지 않는다 — 3.12절 참고).

---

# 3. Database / Storage / Recovery Current State

## 3.1 MariaDB 구성

MariaDB는 2대의 서버로 구성되어 있으며, 한 서버에서 다른 서버로 데이터를 복제하는 구조를 사용한다.

| 구성          | 주소                              | 역할                 |
| ----------- | -------------------------------- | ------------------ |
| mariadb-01  | `192.168.52.40`                  | MariaDB 서버         |
| mariadb-02  | `192.168.51.40`                  | MariaDB 서버         |
| maxscale-01 | `192.168.53.40`                  | MaxScale           |
| DB VIP      | `10.1.93.90:3306`                | 애플리케이션 DB 접속 지점    |
| DB FQDN     | `db.seokpan.soldesk.store:3306`  | 애플리케이션이 사용하는 DB 주소 |

MariaDB 버전은 `11.8.9-log`이며, MaxScale은 `24.02.9`를 사용한다.

Application은 개별 MariaDB 서버의 IP를 직접 사용하는 대신 MaxScale이 제공하는 접속 지점을 사용한다.

```text
Application
    ↓
db.seokpan.soldesk.store:3306
    ↓
10.1.93.90:3306
    ↓
MaxScale
    ↓
MariaDB-01 / MariaDB-02
```

이 구조를 통해 애플리케이션과 실제 MariaDB 서버의 위치를 분리하고, DB 서버 장애가 발생했을 때 MaxScale이 제공하는 장애 전환 구조를 사용할 수 있다.

**⚠️ 중요**: `auto_failover=true`, `auto_rejoin=true`가 실제로 켜져 있어 Master/Replica 역할은 고정이 아니다. 모든 DB 쓰기 작업(계정 생성, 스키마 변경, 백업, 복구 등) 전에는 반드시 `maxctrl list servers`로 현재 Master를 재확인한다.

---

## 3.2 Database Schema

애플리케이션 데이터베이스는 `stone_game`을 사용한다.

주요 테이블은 다음과 같다.

```text
member
member_stats
game
game_participant
move
game_result
rating_history
```

이 구조는 회원 정보와 게임방, 게임 참가자, 게임 진행 기록 및 결과 등의 데이터를 저장하기 위한 것이다.

---

## 3.3 Database Account

Database 계정은 용도에 따라 분리한다.

| 계정                 | 용도                        |
| ------------------ | ------------------------- |
| `identity_svc`     | 회원 및 인증 관련 Application 접근 |
| `game_svc`         | 게임 서비스 Application 접근     |
| `db_admin`         | Schema Migration 등 관리 작업  |
| `backup_svc`       | Backup 작업                 |
| `repl_user`        | MariaDB 서버 간 데이터 복제       |
| `maxscale_monitor` | MaxScale의 DB 상태 확인        |
| `exporter_svc`     | mysqld_exporter Metric 수집(모니터링 전용, `SLAVE MONITOR` 포함 필요 최소 권한, 쓰기 권한 없음) |

Application Runtime 계정과 Migration 계정을 분리하여 애플리케이션이 일반적인 DB 작업을 수행하는 과정에서 Schema 변경 권한까지 직접 사용할 필요가 없도록 구성한다.

## 3.3-1 DB/MaxScale/NFS 서버 관측성(Observability Exporter)

mariadb-01/02, maxscale-01, nfs 4대에는 서버 자체 자원(CPU/메모리/
디스크/네트워크) 수집을 위한 node_exporter(`1.12.1-distroless`)가 배포되어 있다.

mariadb-01/02에는 추가로 MariaDB 서비스 자체 상태(쿼리/복제 통계) 수집을
위한 mysqld_exporter(`0.20.0`)가 `exporter_svc` 계정으로 배포되어 있으며, 배포 검증 중 발견한 `SLAVE MONITOR` 권한 누락 버그는 수정 완료했다(TS-043). Prometheus Scrape 연동은 확인 완료됐으나 Alert Rule 등록은 팀 결정으로 보류된 상태다.

maxscale-01의 MaxScale 서비스 상태(라우팅/Failover) 수집용
maxscale_exporter는 REST read-only 계정과 Role 골격까지만 코드화됐고, 실제 바이너리 배포는 2차 프로젝트로 이관됐다. 현재도 `maxctrl list
servers` 수동 확인에 의존한다.

NFS는 별도 Exporter 없이 node_exporter의 `nfsd` Collector로 대체한다.

```text
서버 자체 자원 → node_exporter (4대 전체)
MariaDB 서비스 상태 → mysqld_exporter (배포·Prometheus 수집 완료, Alert Rule 2차)
MaxScale 서비스 상태 → maxscale_exporter (2차 이관, REST read-only 계정만 코드화)
```

---

## 3.4 Database Credential 정합성

Database 인증 정보는 Vault를 기준으로 관리하며, 실제 MariaDB 계정 및 Kubernetes Secret과 값이 서로 다르게 존재하지 않도록 정합성을 확인했다.

특히 다음 관계를 검증 대상으로 사용한다.

```text
Vault
 ↓
Kubernetes Secret
 ↓
Application
 ↓
MariaDB
```

비밀번호와 같은 실제 Credential 값은 Repository, Issue, PR 및 Evidence에 기록하지 않는다.

---

## 3.5 MariaDB TLS

Application이 Kubernetes 외부의 MariaDB에 연결할 때 TLS를 사용한다.

Kubernetes Workload에는 Root CA를 제공하고, Application이 DB 서버 인증서를 검증할 수 있도록 구성한다.

```text
Application Pod
    ↓
Root CA
    ↓
TLS handshake
    ↓
MaxScale / MariaDB
```

따라서 DB 접속 정보에는 단순히 주소와 계정뿐 아니라 **접속 대상의 인증서를 검증할 수 있는 Root CA**가 함께 필요하다.

---

## 3.6 NFS Storage

NFS 서버는 다음 주소에서 제공한다.

```text
192.168.54.50
```

주요 Export는 목적에 따라 분리한다.

| 경로                   | 용도                             |
| -------------------- | ------------------------------ |
| `/srv/nfs/k8s`       | Kubernetes PersistentVolume 제공 |
| `/mnt/nfs-db-backup` (마운트, 서버 측 export는 db-backup 계열) | MariaDB Backup Chain 공유 상태 관리(`.state/` 서브디렉터리에 `.backup_chain_state.json`) |

NFS를 사용하는 이유는 여러 서버 또는 Kubernetes Pod에서 동일한 저장 공간에 접근할 수 있도록 하기 위함이다.

**경로 혼동 주의**: `/srv/nfs/db-backup`은 각 MariaDB 호스트의 **로컬** 스테이징 디렉터리이며(이름과 달리 NFS 공유 경로가 아님), 실제 NFS 공유 상태 authority는 `/mnt/nfs-db-backup/.state/`다. 이 구분은 DR-01 실측(이슈 #194) 중 실제로 혼동이 발견되어 정정된 사항이다.

---

## 3.7 Kubernetes NFS Storage

Kubernetes에서는 NFS Subdir External Provisioner를 사용하여 PersistentVolume을 동적으로 생성한다.

구성은 다음과 같다.

```text
Kubernetes PVC
    ↓
StorageClass
    ↓
NFS Subdir External Provisioner
    ↓
NFS Server
    ↓
/srv/nfs/k8s
```

이를 통해 Pod가 재생성되더라도 PersistentVolume에 저장된 데이터가 유지될 수 있도록 구성한다.

---

## 3.8 MariaDB Backup

MariaDB Backup은 Replica를 기준으로 수행한다.

Backup 구조는 다음과 같다.

```text
MariaDB Replica
    ↓
Full Backup
    ↓
Incremental Backup
    ↓
NFS Backup Storage
```

Backup은 Full Backup과 Incremental Backup을 연결하여 관리한다.

### Backup Chain 관리

Incremental Backup은 이전 Backup과 연결된 상태를 확인한 후 생성한다.

정상적인 Chain이 존재하지 않으면 무효한 Incremental Backup을 계속 연결하지 않고 Full Backup부터 새로운 Chain을 시작한다.

또한 MaxScale 장애 전환으로 Backup을 실행하는 DB 서버가 변경될 수 있기 때문에 Backup Chain 상태를 특정 DB 서버의 Local 파일에만 저장하지 않고 NFS에 공유한다(3.6절 경로 참고).

---

## 3.9 Backup 동시 실행 보호

Backup 작업이 여러 서버에서 동시에 실행될 경우 동일한 Chain 상태를 동시에 수정하면 Backup 상태가 꼬일 수 있다.

이를 방지하기 위해 NFS에 공유 Lock을 사용한다.

```text
Backup 실행 요청
    ↓
NFS Shared Lock 획득
    ↓
Backup Chain 상태 확인
    ↓
Backup 실행
    ↓
Chain 상태 갱신
    ↓
Lock 해제
```

동시 실행 테스트를 통해 여러 Backup 작업이 동시에 Chain 상태를 수정하지 않는 것을 확인했다(3회 반복, 동시 획득 0건).

---

## 3.10 Backup 보존 및 무결성

Backup은 다음 정책을 사용한다.

* Full + Incremental Chain
* 7일 보존
* SHA-256 기반 무결성 확인
* GTID 기반 복구 위치 확인
* 유효한 Chain이 없을 경우 Full Backup으로 재시작

이 구조는 단순히 Backup 파일을 보관하는 것이 아니라 **복구 가능한 Backup Chain인지 확인하는 것**을 목표로 한다.

---

## 3.11 MariaDB Recovery

MariaDB 복구는 `mariadb_dr_recovery.yml`을 이용하여 수행한다.

전체 흐름은 다음과 같다.

```text
Backup Chain 확인
    ↓
Full Backup Prepare
    ↓
Incremental Backup 순차 Prepare
    ↓
복구 데이터 구성
    ↓
MariaDB 기동
    ↓
Replication 재구성
    ↓
MaxScale 상태 확인
    ↓
서비스 접근 확인
    ↓
데이터 정합성 검증
```

복구 과정에서는 단순히 MariaDB 프로세스가 실행되는지만 확인하지 않는다.

다음 항목을 함께 확인한다.

* 복구된 데이터 건수
* 데이터 무결성
* 외래키 관계
* GTID 상태
* Replication 상태
* Application 계정 접근 가능 여부
* MaxScale 상태
* 서비스 접근 가능 여부

Recovery는 `backup_restore_mode: isolated`(격리 인스턴스 검증용)와 `production`(실제 서비스 datadir 대상)을 명시적으로 분기한다. production 모드에서는 Count/CHECKSUM Gate가 `assert` 대신 정보성 리포트로 전환되며(백업 시점과 복구 시점 사이 정상적인 데이터 간극이 존재할 수 있으므로), SHA-256 무결성 검증은 두 모드 모두 하드 Gate로 유지한다. 양쪽 다 유실된 최악 시나리오에서는 MaxScale의 Split-brain 위험(고립된 두 Master 동시 존재)을 막기 위해 role=master 확립 시 나머지 호스트에 `read_only=1` 강제 배포 안전장치가 자동 적용·자동 해제된다.

---

## 3.12 MariaDB DR-01 최종 실측 (완료, 이슈 #194 — 2026-09-18 close)

2026-09-16 DR-01 실측에서는 실제 복구 절차를 실행하여 RTO와 RPO를 측정했다. Isolated 모드(격리 인스턴스, 검증 목적)와 Production 모드(양쪽 서버 동시 유실 최악 시나리오, 실제 서비스 datadir 대상)를 구분해서 각각 측정했다.

### 최종 측정값 (확정, 09/12 문서 동일하게 사용)

| 시나리오 | 조건 | RTO | RPO |
| --- | --- | --- | --- |
| Isolated | `mariadb_restore_chain.yml`, mariadb-02 대상, Full-only 체인(`chain_20260916`) | **27.376초** | 해당 없음(격리 검증용, 실제 데이터 유실 시나리오 아님) |
| Production | 양쪽 서버 동시 정지 → `mariadb_dr_recovery.yml` 4단계를 master→replica 순 완주 | 단일 노드 재개 **1분 29초** / 양쪽 이중화 완전 정상화 **4분 0초** | **3건 손실**(더미데이터 B 전량, Incremental 이후 미백업 데이터가 설계대로 정확히 손실) |

Isolated 모드는 Count/CHECKSUM/FK Gate 7개 테이블·7개 관계 전부 PASS. Production 모드는 복구 후 mariadb-01 vs mariadb-02 Count/CHECKSUM 7개 테이블 전부 일치, 그리고 3.11절의 Split-brain 방지 안전장치(read_only 강제 배포)가 실제로 mariadb-02 재편입 시점에 자동 해제되는 것까지 이때 최초로 실측 검증됐다.

> **주의**: 이 두 시나리오의 RTO/RPO는 서로 다른 조건(격리 검증 vs 실제 유실 복구)에서 나온 값이므로 하나의 수치로 합치거나 서로 대체하지 않는다. 두 값 모두 위 표를 최종 값으로 사용하며, 이전에 이 절이 "최종 실측값 반영 필요"로 표시했던 것은 이 표로 대체됐다.

DR-01 실측 과정에서는 `roles/backup_transfer/tasks/replication_setup.yml:117`("복제 정상 기동 Gate" 태스크)에 `when: backup_restore_role == 'replica'` 조건이 누락되어 있어, `backup_restore_role=master` 경로 실행 시 register되지 않은 `dr_slave_status.query_result`를 참조하다 fatal 처리되는 버그가 발견되었다(같은 파일의 다른 3개 태스크는 이미 정상적으로 조건이 걸려 있었음). PR #197에서 조건을 추가해 수정했으며, 수정 후 Master/Replica 양쪽에서 재검증해 두 경로 모두 `failed=0`으로 정상 완료되는 것을 확인했다(리뷰어 이유빈, TS-042로 게시).

같은 실측 중 3.6절의 NFS 공유 경로 표기 오류(`/mnt/nfs-db-backup` 루트가 아니라 `.state/` 서브디렉터리)도 함께 발견·정정됐다.

---

## 3.13 Redis

Redis는 Kubernetes `platform` namespace에서 StatefulSet으로 운영한다.

```text
Application
    ↓
redis.platform.svc.cluster.local:6379
    ↓
Redis StatefulSet
    ↓
Redis PVC
    ↓
NFS
```

Redis는 AOF(Append Only File)를 사용하며 `appendfsync everysec` 설정으로 데이터를 영속화한다.

검증 결과 다음 항목을 확인했다.

* Redis Write / Read 정상 동작
* Pod 재생성 후 데이터 유지
* PVC 기반 데이터 보존

따라서 **Redis 기본 영속성은 검증된 상태**이다(seokpan-gitops#7).

### Redis DR-03 — PVC 손상 시 MariaDB 기준 재구성 (1차 Infra 레벨 검증 완료, 이슈 #115, 2026-09-18)

Redis DR은 "Redis 데이터를 별도로 백업했다가 복원"하는 방식이 아니라, **MariaDB의 확정 기록(move, game_result, rating_history)이 항상 진실이고, Redis PVC가 손상되면 MariaDB 기준으로 Redis 상태를 재구성한다**는 설계다(`F_redis_recovery_contract.md`). Room/현재 투표/Ready 상태 같은 "지금 이 순간"의 공유 런타임 상태만 Redis가 담당하고, 그중에서도 아직 MariaDB에 확정되지 않은 진행 중 투표는 복구 대상이 아니다(유실돼도 정상 동작으로 인정).

동일 스펙(redis:8.10.1, appendonly yes/everysec, nfs-k8s PVC)의 격리 StatefulSet(`storage-infra/redis-dr03-test`)에서 실제 운영 데이터 기반 Key Schema를 확인하고 다음을 검증했다.

* 정상 재기동 시 AOF Replay로 완전 복구(PASS)
* AOF(base.rdb) 직접 손상 시 CrashLoopBackOff 재현(PASS, "Wrong RDB checksum ... RDB CRC error")
* MariaDB `move` 테이블(`move_no` 순) 기준 SQL 수동 재구성 → 사전 계산값과 완전 일치(PASS)
* Redis Ahead(무효화)/Behind(재동기화)/Exact 3케이스 판정 로직(PASS)
* Data Loss/Duplicate/Stale 3종 무결성 검증(PASS, 전부 없음)

**이 검증 완료가 곧 운영 반영 완료를 의미하지 않는다.** 운영 `platform/redis-0`에 대한 자동 복구(Ansible 자동화, `redis-dr-recovery-automation-handoff.md`로 인계)는 아직 착수 전이며, 재구성값을 운영 Redis에 실제로 쓰는 권한 설계가 BLOCKING 선행조건으로 남아 있다. 장애 관찰/주입용 권한(`pods:delete`, `pods/log:get`)은 `platform/ksh` SA에 이미 부여됐고(seokpan-gitops#47/#48), `pods/exec`는 부여하지 않기로 확정했다. 이슈 #115는 이 잔여 작업 때문에 open 상태를 유지한다.

---

## 3.14 etcd DR

Kubernetes의 클러스터 상태 데이터는 etcd Snapshot을 이용해 보호한다.

```text
Kubernetes etcd
    ↓
Snapshot
    ↓
SHA-256 검증
    ↓
격리된 3-member etcd 복구 환경
    ↓
Quorum 확인
    ↓
kube-apiserver 인증 확인
    ↓
Kubernetes Object 비교
```

**진행 이력(3단계로 구분)**:

```text
이슈 #113 (2026-09-08 스코프 축소, completed)
  → Snapshot 생성 + SHA-256 무결성 검증 + NFS 전용 경로(/srv/nfs/etcd-dr) 전송까지만

이슈 #156 (2026-09-11, closed) — 1차 수동 E2E
  → 별도 물리PC 격리망(loadgen/loadgen2/loadgen3, VMnet 192.168.55.0/24)에서
    수동으로 3-member Restore → K8s API 기동 → Object 검증 → RTO 측정
  → RTO = 38분 44초(트러블슈팅 대응 시간 포함), Namespace 12/Deployment 21/Secret 37 diff 0

이슈 #176 + PR #191 (2026-09-14~16, 2026-09-16 01:32 UTC merged) — 자동화
  → Safety Guard → Restore Decision Gate → Final Restore Gate 3단 안전장치 자동화
  → bridged loadgen 환경(10.1.93.95~97/24)에서 E2E 자동 실행
```

격리 환경에서 3-member etcd를 복구하고 다음 항목을 확인했다.

* etcd Leader / Raft 상태
* Snapshot Hash / Revision
* 3-member Quorum
* kube-apiserver 인증
* Namespace / Deployment / Secret Object 수
* 원본과 복구 결과 비교(Namespace 13→13, Deployment 21→21, Secret 39→39, diff 0)

**최종 DR-02 E2E 검증(자동화, PR #191)에서는 RTO 52초**(Restore→Quorum 21초 / Quorum→API 27초 / API→Object 4초)가 측정되었다. 09/12 문서에서 "etcd DR-02 RTO"를 대표값으로 인용할 때는 이 52초를 사용하고, #156의 수동 측정값(38분44초)은 자동화 이전 참고값으로만 병기한다.

이 결과는 **etcd Snapshot 기반 Kubernetes 상태 복구 시간**을 의미하며, 전체 Kubernetes 클러스터 재구축 시간이나 Worker Node 재가입 시간을 포함한 전체 재해복구 시간으로 해석하지 않는다.

---

# 4. Provider / Consumer Contract

## 4.1 Database Provider

| Provider | 제공 내용                 | Consumer                 |
| -------- | --------------------- | ------------------------ |
| MaxScale | DB 접속 Endpoint        | Backend                  |
| MariaDB  | `stone_game` Database | Backend / Migration      |
| Vault    | Credential            | Kubernetes / Application |
| Root CA  | DB 인증서 검증 정보          | Backend                  |
| NFS      | Backup 저장소            | Backup / Recovery        |

---

## 4.2 Database Consumer

### Backend Runtime

Backend는 다음 정보를 이용하여 MariaDB에 연결한다.

```text
DB_HOST
DB_PORT
DB_NAME
DB_USER
DB_PASSWORD
DB_CA
```

실제 접속 경로는 다음과 같다.

```text
Backend Pod
    ↓
db.seokpan.soldesk.store:3306
    ↓
MaxScale VIP
    ↓
MariaDB
```

---

## 4.3 Migration Consumer

Migration은 Runtime 계정과 별도의 관리 계정을 사용한다.

```text
Migration Job
    ↓
db_admin
    ↓
MariaDB
    ↓
Schema 변경
```

Migration은 Application Runtime이 시작되기 전에 필요한 Schema가 준비되어 있는지 확인하는 역할을 한다.

---

## 4.4 Redis Consumer

Backend는 Kubernetes Service를 통해 Redis에 접근한다.

```text
Backend Pod
    ↓
redis.platform.svc.cluster.local:6379
    ↓
Redis
    ↓
Game Room / Runtime State
```

MariaDB가 회원 및 영속적인 게임 데이터를 저장한다면 Redis는 실시간 게임 상태와 같이 빠른 접근이 필요한 데이터를 저장하는 역할을 수행한다. 두 저장소 간 상태 일치 판정과 재구성 규칙은 3.13절 Redis DR-03을 따른다.

---

# 5. Integration 흐름

## 5.1 정상 Application → MariaDB 흐름

```text
User Request
    ↓
Frontend
    ↓
Backend
    ↓
DB FQDN
    ↓
MaxScale
    ↓
MariaDB
    ↓
stone_game
```

Backend가 DB에 직접 접속하는 것이 아니라 MaxScale을 통해 접속하기 때문에 DB 서버의 실제 위치와 Application을 분리할 수 있다.

---

## 5.2 Application → Redis 흐름

```text
Game Request
    ↓
Backend
    ↓
redis.platform.svc.cluster.local
    ↓
Redis
    ↓
Game State
```

게임방과 같은 실시간 상태는 Redis에서 빠르게 조회하고 변경한다.

---

## 5.3 Backup 흐름

```text
MariaDB Replica
    ↓
Backup Automation
    ↓
Full / Incremental
    ↓
NFS
    ↓
/mnt/nfs-db-backup/.state/ (상태 authority) + 각 호스트 로컬 스테이징(/srv/nfs/db-backup)
```

Backup Chain 상태도 NFS에 함께 관리하여 MariaDB 서버가 변경되어도 기존 Chain의 상태를 이어서 관리할 수 있도록 한다.

---

## 5.4 MariaDB Recovery 흐름

```text
NFS Backup
    ↓
Backup Chain 확인
    ↓
Full Prepare
    ↓
Incremental Prepare
    ↓
MariaDB Restore
    ↓
Replication 재구성
    ↓
MaxScale 확인
    ↓
Application 계정 접속 확인
    ↓
데이터 정합성 확인
```

---

## 5.5 Redis DR-03 흐름 (PVC 손상 시)

```text
Redis PVC 손상 감지
    ↓
MariaDB move 테이블 조회(game_id, move_no 순)
    ↓
보드/턴/현재 팀 재구성
    ↓
Redis Ahead/Behind/Exact 판정
    ↓
필요 시 MariaDB 기준 강제 재동기화
    ↓
(운영 반영은 자동화 착수 전 — §3.13 BLOCKING 참고)
```

---

## 5.6 Kubernetes etcd Recovery 흐름

```text
etcd Snapshot
    ↓
Hash / Revision 확인
    ↓
Safety Guard → Restore Decision Gate → Final Restore Gate
    ↓
격리 환경 Restore
    ↓
3-member Quorum 확인
    ↓
kube-apiserver 연결
    ↓
Kubernetes Object 비교
```

---

# 6. Integration Gate

## Gate A — Database Provider Ready

### 확인 내용

* MariaDB 2대 구성 확인
* MaxScale 동작 확인
* DB VIP / FQDN 접속 확인
* `stone_game` Database 확인
* Runtime / Migration 계정 확인

### 판정

```text
MariaDB + MaxScale + Schema + Account
→ 정상 확인
→ PASS
```

---

## Gate B — Database Security Contract

### 확인 내용

* Vault Credential 확인
* Kubernetes Secret과 Credential 정합성 확인
* Root CA 제공 확인
* MariaDB TLS 연결 확인

### 판정

```text
Vault
→ Kubernetes Secret
→ Application
→ MariaDB TLS
→ PASS
```

---

## Gate C — NFS / Storage Provider Ready

### 확인 내용

* NFS 서버 접근
* Kubernetes StorageClass
* PVC 생성
* Pod 재생성 후 데이터 유지
* DB Backup 경로 접근

### 판정

```text
NFS
→ Kubernetes Storage
→ PVC
→ Persistence
→ PASS
```

---

## Gate D — MariaDB Backup Ready

### 확인 내용

* Full Backup 생성
* Incremental Backup 생성
* Backup Chain 연결
* SHA-256 검증
* 7일 Retention
* NFS 공유 상태
* Shared Lock 동시 실행 방지

### 판정

```text
Backup 생성
→ Chain 유지
→ 무결성 확인
→ 공유 저장
→ PASS
```

---

## Gate E — MariaDB Recovery Ready

### 확인 내용

* Backup Chain Restore
* Full / Incremental Prepare
* MariaDB 기동
* 데이터 정합성 확인
* Replication 재구성
* MaxScale 상태 확인
* Application 계정 접속 확인
* 최종 RTO / RPO 측정

### 판정

```text
Backup
→ Restore
→ Database 정상화
→ Replication
→ MaxScale
→ Service Check
→ RTO / RPO Evidence
→ PASS
```

**최종 판정: PASS(완료)** — Isolated RTO 27.376초, Production RPO 3건/RTO 1분29초(단일)·4분(이중화). 3.12절 표를 최종 값으로 사용한다.

---

## Gate F — Redis Persistence / Recovery

### 확인 내용

* Redis Write / Read
* PVC 연결
* AOF 활성화
* Pod 재생성
* 기존 데이터 유지
* PVC 손상 시 MariaDB 기준 재구성(DR-03) 절차 검증

### 판정

```text
Redis Persistence
→ PASS

Redis DR-03(1차 Infra 레벨, 격리 환경)
→ PASS

Redis DR-03 운영 자동화
→ Not Tested(BLOCKING: 운영 Pod 쓰기 반영 권한 결정)
```

---

## Gate G — Kubernetes etcd DR

### 확인 내용

* Snapshot 생성
* SHA-256 검증
* 격리 환경 Restore
* 3-member Quorum
* kube-apiserver 인증
* Kubernetes Object 비교
* RTO 측정

### 판정

```text
Snapshot
→ Restore
→ Quorum
→ API
→ Object Validation
→ PASS
```

최종 E2E 측정 RTO는 **52초**(자동화, PR #191)이다. 자동화 이전 수동 측정값(38분44초, 이슈 #156)은 참고값으로만 병기한다.

---

# 7. Gate 요약

| Gate | 대상                       | 현재 상태       | 핵심 Evidence                              |
| ---- | ------------------------ | ----------- | ---------------------------------------- |
| A    | MariaDB / MaxScale       | `Validated` | DB Provider / Schema / Account           |
| B    | Credential / TLS         | `Validated` | Vault ↔ Secret ↔ DB                      |
| C    | NFS / Kubernetes Storage | `Validated` | PVC / Persistence                        |
| D    | MariaDB Backup           | `Validated` | Full / Incremental / Chain / Lock        |
| E    | MariaDB Recovery         | **`Validated`(완료)** | Restore / 정합성 / RTO·RPO 확정(이슈 #194) |
| F    | Redis Persistence        | `Validated` | Write / Read / Pod Recreation            |
| F    | Redis DR-03(1차 Infra)    | **`Validated`** | 격리 환경 재구성 절차 검증 완료(이슈 #115) |
| F    | Redis DR-03(운영 자동화)      | `Not Tested` | BLOCKING: RBAC/쓰기 반영 권한 결정            |
| G    | etcd DR                  | `Validated` | Isolated Restore / Quorum / API / Object, RTO 52초(자동화) |

---

# 8. 남은 Blocker와 Gap

## 8.1 ~~MariaDB Recovery 실측 결과 문서 반영~~ (해결됨, 2026-09-18)

DR-01 실제 복구 실측이 완료되어 3.12절에 최종 RTO/RPO 값을 반영했다. 이전 절이 명시했던 "실측값 반영 필요" 상태는 해소됐다.

```text
초기 측정값(없음, 이번이 최초 실측)
→ 3.12절 표가 최종 확정값
```

---

## 8.2 Redis DR-03 운영 자동화 (잔여, BLOCKING)

현재 Redis는 다음까지 검증되어 있다.

```text
기본 Persistence(Write/Read/PVC/Pod Recreation)
→ 1차 Infra 레벨 DR-03 절차 검증(격리 환경, MariaDB 기준 재구성)
```

하지만 이 절차를 운영 `platform/redis-0`에 자동으로 반영하는 Ansible 자동화는 아직 착수 전이며, 재구성값을 운영 Pod에 실제로 써넣는 권한 설계가 선행되어야 한다(`pods/exec` 미부여 확정, 대안 경로 미정). 따라서 현재 상태를 `Redis DR PASS(운영 반영 포함)`로 확대 해석하지 않고, "1차 Infra 레벨 PASS + 운영 자동화 Not Tested"로 구분해서 유지한다.

---

## 8.3 전체 Kubernetes Disaster Recovery

etcd Snapshot 기반 복구는 검증되었지만 다음 작업은 별도의 범위다.

```text
etcd Restore
≠
Full Kubernetes Cluster Rebuild
```

특히 다음은 별도의 검증 대상이다.

* kubeadm PKI 복구
* Control Plane 재구성
* Worker Node 재가입
* CNI 재구성
* 외부 인프라 복구
* Application 재배포
* 외부 DNS / TLS 복구

---

## 8.4 Application Data Integration

Database Provider와 Recovery 기능이 검증되었다고 해서 Application 전체의 데이터 흐름이 완료된 것은 아니다.

최종적으로 다음 흐름을 Application 수준에서 확인해야 한다.

```text
Browser
→ Frontend
→ Backend
→ MariaDB
→ Redis
```

특히 회원 가입, 게임방 생성, 게임 진행, 결과 저장 등의 실제 사용자 시나리오를 통해 MariaDB와 Redis가 각각 의도한 데이터를 저장하는지 확인할 필요가 있다.

## 8.5 DB/MaxScale 서비스 레벨 Observability 잔여 범위

현재 Prometheus로 수집되는 것은 4대 서버의 자체 자원(node_exporter)과
mariadb-01/02의 MariaDB 서비스 상태(mysqld_exporter)까지다. 남은 잔여
범위는 다음과 같다.

```text
mysqld_exporter
→ 배포·계정·Metric 수집·Prometheus 연동까지 완료
→ Alert Rule(복제 지연, Slave 중단)은 2차 이관

maxscale_exporter
→ 공식 exporter 부재(MXS-3022 Won't Do)로 소스 빌드 방식으로 2차 재착수 (REST read-only 계정만 코드화)

vrouter 방화벽
→ 9100/tcp rich rule이 mariadb-01/02 대역만 확인됨, maxscale-01·nfs 대역 커버 여부 네트워크 담당자 확인 필요
```

MaxScale Failover 발생을 Alert로 조기 탐지하는 경로는 아직 없으며, 장애
인지는 계속 수동 확인(`maxctrl list servers`, 로그 확인)에 의존한다. 이
Gap은 MariaDB DR Recovery 완료 기준(RTO/RPO)과는 별개이며, 별도 팀
결정으로 관리한다.

---

# 9. Critical Path

현재 Database / Storage / Recovery 영역의 최종 Critical Path는 다음과 같다.

```text
NFS / Storage
    ↓
MariaDB / MaxScale
    ↓
Credential / TLS
    ↓
Backup Chain
    ↓
MariaDB Recovery (완료)
    ↓
RTO / RPO Evidence (확정)
    ↓
Redis Persistence
    ↓
Redis DR-03 (1차 완료, 운영 자동화 잔여)
    ↓
etcd DR (완료, RTO 52초)
    ↓
Application Data Consumer
    ↓
MVP Data Platform Acceptance
```

### MariaDB Recovery 핵심 경로 (완료)

```text
Full Backup
    ↓
Incremental Chain
    ↓
NFS Backup
    ↓
Restore Chain
    ↓
MariaDB Recovery
    ↓
Replication
    ↓
MaxScale
    ↓
Application Account
    ↓
Data Integrity
    ↓
RTO / RPO (확정)
```

이 흐름은 이슈 #194로 실제 실행까지 완료됐다. 남은 것은 Redis DR-03 운영 자동화와 Application Data Consumer의 최신 Revision 기준 재검증이다.

---

# 10. 완료 기준

## 10.1 Database

* [x] MariaDB 2대 구성
* [x] MaxScale 구성
* [x] DB VIP / FQDN 구성
* [x] `stone_game` Schema 구성
* [x] Runtime / Migration 계정 분리
* [x] MariaDB TLS
* [x] Vault Credential 정합성 확인

---

## 10.2 Storage

* [x] NFS Server 구성
* [x] Kubernetes StorageClass 구성
* [x] PVC 기반 Persistence 검증
* [x] MariaDB Backup 저장 경로 구성
* [x] Backup Chain 공유 상태 구성
* [x] Shared Lock 검증

---

## 10.3 MariaDB Backup / Recovery

* [x] Full Backup
* [x] Incremental Backup
* [x] Backup Chain
* [x] SHA-256 무결성 검증
* [x] 7일 Retention
* [x] GTID 기반 복구
* [x] Backup Restore
* [x] Replication 재구성
* [x] 데이터 정합성 검증
* [x] MaxScale 상태 검증
* [x] Application 계정 접근 검증
* [x] DR-01 실측
* [x] 최종 RTO / RPO 측정 — **Isolated 27.376초 / Production RPO 3건·RTO 1분29초(단일)·4분(이중화)**(이슈 #194, 2026-09-18 close)

---

## 10.4 Redis

* [x] Redis StatefulSet
* [x] PVC 연결
* [x] AOF Persistence
* [x] Write / Read
* [x] Pod Recreation 후 데이터 유지
* [x] DR-03 1차 Infra 레벨 검증(격리 환경, MariaDB 기준 재구성, 이슈 #115)
* [ ] DR-03 운영 자동화(Ansible, `platform/redis-0` 실제 쓰기 반영) — BLOCKING: 쓰기 권한 결정

---

## 10.5 etcd DR

* [x] etcd Snapshot
* [x] Snapshot Hash / Revision
* [x] NFS 전달
* [x] 격리된 3-member Restore
* [x] Quorum 확인
* [x] kube-apiserver 인증
* [x] Kubernetes Object 비교
* [x] E2E RTO 측정(수동 1차 + 자동화)

최종 측정 결과:

```text
etcd DR-02 RTO(수동, 이슈 #156, 참고값) = 38분 44초
etcd DR-02 RTO(자동화, 이슈 #176/PR #191, 대표값) = 52초
```

---

# 11. Traceability

## 11.1 Database / Backup / Recovery

| 항목               | 추적 대상           | 목적                                    |
| ---------------- | --------------- | -------------------------------------- |
| DB 기본 구성         | `seokpan-infra` | MariaDB / MaxScale 구성                 |
| Backup Chain     | `seokpan-infra` | Full / Incremental Backup             |
| Backup 공유 상태     | `seokpan-infra` | NFS 기반 Chain 상태 공유                    |
| Backup 동시 실행 보호  | `seokpan-infra` | Shared Lock                           |
| MariaDB Recovery | `seokpan-infra` | Restore / Replication / Service Check |
| DR-01 실측(완료)     | Infra #194(closed, 2026-09-18) | MariaDB RTO/RPO 실측 및 확정        |
| Recovery 버그 수정   | Infra PR #197   | `replication_setup.yml:117` Gate `when` 누락 수정 |

---

## 11.2 Storage

| 항목                 | 추적 대상                 | 목적                    |
| ------------------ | --------------------- | --------------------- |
| NFS Server         | Infra #52             | NFS Storage 구성        |
| NFS Provisioner    | Infra PR #61 / PR #78 | Kubernetes Storage 제공 |
| Kubernetes Storage | GitOps #10 / PR #13   | StorageClass / PVC 구성 |

---

## 11.3 Credential / Security

| 항목                | 추적 대상                | 목적                                 |
| ----------------- | -------------------- | ---------------------------------- |
| DB Credential 정합성 | Infra #172 / PR #172 | Vault / DB / Kubernetes Secret 정합성 |
| Replication       | Infra #197           | Master 경로 fatal 버그 수정 검증           |

> **정정**: 이전 버전에서 이 표의 "Recovery Safety" 항목 근거로 인용했던 Infra #176은 실제로는 **etcd DR-02 자동화 전용 이슈**(Safety Guard/Restore Decision Gate)이며 MariaDB Recovery와는 무관하다. MariaDB 쪽의 잘못된 환경 실행 방지는 `mariadb_dr_recovery.yml`의 `backup_restore_mode`(isolated/production) 명시적 분기(3.11절)로 구현되어 있다.

---

## 11.4 Kubernetes DR

| 항목            | 추적 대상             | 목적                            |
| ------------- | ----------------- | ------------------------------ |
| etcd DR 설계    | Infra #113        | Snapshot / 무결성 / NFS 전송(축소 스코프) |
| etcd DR-02 수동 E2E | Infra #156(closed) | 격리 환경 Restore 및 RTO 수동 측정(38분44초) |
| etcd DR-02 자동화 | Infra #176(closed), PR #191(merged) | Safety Guard/Restore Decision Gate 자동화, RTO 52초 |
| DR 보호 범위      | Infra #143 계열      | 전체 DR 범위 및 MVP 범위 관리          |

---

## 11.5 Redis DR-03

| 항목                | 추적 대상          | 목적                                |
| ----------------- | -------------- | ---------------------------------- |
| DR-03 설계/계약        | `F_redis_recovery_contract.md` | MariaDB 기준 재구성 규칙 정의     |
| DR-03 1차 검증        | Infra #115(open) | 격리 환경(redis-dr03-test) 재구성 절차 검증 |
| DR-03 RBAC         | seokpan-gitops#47 / #48 | 장애 관찰/주입 권한(`pods:delete`+`pods/log:get`) 부여, `pods/exec` 미부여 확정 |
| DR-03 운영 자동화 인계    | `redis-dr-recovery-automation-handoff.md` | Ansible 자동화 착수 명세(진행 중) |

---

## 11.6 최종 판정

본 문서에서 중요한 판단 기준은 **구현 여부가 아니라 실제 검증 여부**이며, 나아가 **"격리/1차 검증 완료"와 "운영 반영 완료"도 서로 다른 판정**임을 구분한다.

```text
구성 완료
    ↓
실행
    ↓
검증(격리/1차)
    ↓
측정
    ↓
Evidence
    ↓
운영 반영(해당하는 경우)
    ↓
Gate 판정
```

따라서 MariaDB Backup·Recovery(완료), etcd DR(완료), Redis Persistence(완료), Redis DR-03(1차 완료·운영 자동화 잔여)은 각각 독립적으로 검증 상태를 관리하며, 하나의 구성요소가 정상이라고 해서 전체 Data Platform Recovery가 완료된 것으로 판단하지 않는다.

최종 MVP Data Platform의 완료 조건은 다음과 같다.

```text
MariaDB
+ MaxScale
+ Credential / TLS
+ NFS Storage
+ Backup
+ MariaDB Recovery (완료)
+ RTO / RPO Evidence (확정)
+ Redis Persistence
+ Redis DR-03 운영 자동화 (잔여, BLOCKING)
+ etcd DR (완료)
+ Application Data Consumer 검증
```

이 모든 항목의 실제 실행 결과와 Evidence를 기준으로 최종 상태를 판정하며, Redis DR-03 운영 자동화가 남아 있는 한 "전체 Data Platform DR Acceptance 완료"는 아직 선언하지 않는다.