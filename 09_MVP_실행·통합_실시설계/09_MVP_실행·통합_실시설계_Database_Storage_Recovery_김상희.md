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
| ---------------- | ----------------------------------------------------------------- | ------------------------- |
| MariaDB          | mariadb-01 / mariadb-02                                           | 데이터베이스 운영 및 복제 상태 확인      |
| MaxScale         | maxscale-01                                                       | DB 접속 지점과 장애 전환 확인        |
| Database Schema  | `stone_game`                                                      | 애플리케이션 데이터 구조 확인          |
| Database Account | `identity_svc`, `game_svc`, `db_admin`, `backup_svc`, `repl_user` | 용도별 접근 권한 및 인증 확인         |
| TLS              | MariaDB TLS + Root CA                                             | Kubernetes → DB 암호화 연결 확인 |
| NFS              | `192.168.54.50`                                                   | 백업 및 Kubernetes 저장소 제공    |
| MariaDB Backup   | Full + Incremental                                                | 백업 체인 생성 및 보존 확인          |
| MariaDB Recovery | `mariadb_dr_recovery.yml`                                         | 백업 기반 DB 복구 및 RTO/RPO 측정  |
| Redis            | `platform` namespace                                              | 게임 상태 데이터 저장 및 영속성 확인     |
| etcd DR          | Kubernetes etcd Snapshot                                          | Kubernetes 상태 데이터 복구 확인   |

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

---

# 3. Database / Storage / Recovery Current State

## 3.1 MariaDB 구성

MariaDB는 2대의 서버로 구성되어 있으며, 한 서버에서 다른 서버로 데이터를 복제하는 구조를 사용한다.

| 구성          | 주소                              | 역할                 |
| ----------- | ------------------------------- | ------------------ |
| mariadb-01  | `192.168.52.40`                 | MariaDB 서버         |
| mariadb-02  | `192.168.51.40`                 | MariaDB 서버         |
| maxscale-01 | `192.168.53.40`                 | MaxScale           |
| DB VIP      | `10.1.93.90:3306`               | 애플리케이션 DB 접속 지점    |
| DB FQDN     | `db.seokpan.soldesk.store:3306` | 애플리케이션이 사용하는 DB 주소 |

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
| `exporter_svc` | mysqld_exporter Metric 수집(모니터링 전용, 읽기 전용) |

Application Runtime 계정과 Migration 계정을 분리하여 애플리케이션이 일반적인 DB 작업을 수행하는 과정에서 Schema 변경 권한까지 직접 사용할 필요가 없도록 구성한다.

## 3.3-1 DB/MaxScale/NFS 서버 관측성(Observability Exporter)

mariadb-01/02, maxscale-01, nfs 4대에는 서버 자체 자원(CPU/메모리/
디스크/네트워크) 수집을 위한 node_exporter가 배포되어 있다.

mariadb-01/02에는 추가로 MariaDB 서비스 자체 상태(쿼리/복제 통계) 수집을
위한 mysqld_exporter가 `exporter_svc` 계정으로 배포되어 있으나,
Prometheus Scrape 연동과 Alert Rule 등록은 팀 결정으로 보류된 상태다.

maxscale-01의 MaxScale 서비스 자체 상태(라우팅/Failover) 수집용
maxscale_exporter는 구현 자체가 보류 상태이며, 현재도 `maxctrl list
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
| `/srv/nfs/db-backup` | MariaDB Backup 저장 및 공유 상태 관리   |

NFS를 사용하는 이유는 여러 서버 또는 Kubernetes Pod에서 동일한 저장 공간에 접근할 수 있도록 하기 위함이다.

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

또한 MaxScale 장애 전환으로 Backup을 실행하는 DB 서버가 변경될 수 있기 때문에 Backup Chain 상태를 특정 DB 서버의 Local 파일에만 저장하지 않고 NFS에 공유한다.

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

동시 실행 테스트를 통해 여러 Backup 작업이 동시에 Chain 상태를 수정하지 않는 것을 확인했다.

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

---

## 3.12 MariaDB DR-01 최종 실측

2026-09-16 DR-01 실측에서는 실제 복구 절차를 실행하여 RTO와 RPO를 측정했다.

실측은 Full + Incremental Backup Chain을 이용한 특정 시점 복구를 기준으로 수행했으며, 복구 과정의 각 단계와 최종 서비스 복구 상태를 확인했다.

### 최종 측정값

| 항목        | 최종 실측 결과                                  |
| --------- | ----------------------------------------- |
| RTO       | **최종 실측값 반영 필요**                          |
| RPO       | **최종 실측값 반영 필요**                          |
| Backup 방식 | Full + Incremental Chain                  |
| 복구 방식     | Backup Restore + Recovery 절차              |
| 정합성 검증    | GTID / Replication / 데이터 / Application 계정 |
| 추가 검증     | MaxScale 및 서비스 상태                         |

> **주의:** 이전에 사용했던 초기 RTO 측정값은 최종 결과가 아니므로 본 문서에서는 사용하지 않는다.

DR-01 실측 과정에서는 실제 환경에서 복구 절차를 수행하면서 `master` 경로에서 Replica 전용 검증 태스크가 실행되는 문제가 발견되었다. 해당 문제는 PR #197에서 `backup_restore_role=replica` 조건을 추가하여 수정했으며, 수정 후 Master/Replica 양쪽에서 재검증하여 두 경로 모두 `failed=0`으로 정상 완료되는 것을 확인했다.

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

따라서 **Redis 기본 영속성은 검증된 상태**이다.

다만 Redis 데이터를 권한 있는 Backup으로 별도 보호하고, 장애 상황에서 해당 Backup을 이용해 복구하는 **완전한 Redis DR 검증은 아직 완료되지 않았다.**

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

격리 환경에서 3-member etcd를 복구하고 다음 항목을 확인했다.

* etcd Leader / Raft 상태
* Snapshot Hash / Revision
* 3-member Quorum
* kube-apiserver 인증
* Namespace / Deployment / Secret Object 수
* 원본과 복구 결과 비교

최종 DR-02 E2E 검증에서는 **RTO 52초**가 측정되었다.

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

MariaDB가 회원 및 영속적인 게임 데이터를 저장한다면 Redis는 실시간 게임 상태와 같이 빠른 접근이 필요한 데이터를 저장하는 역할을 수행한다.

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
/srv/nfs/db-backup
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

## 5.5 Kubernetes etcd Recovery 흐름

```text
etcd Snapshot
    ↓
Hash / Revision 확인
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

단, 최종 RTO/RPO 수치는 실측 결과 문서와 동일한 값을 사용해야 한다.

---

## Gate F — Redis Persistence / Recovery

### 확인 내용

* Redis Write / Read
* PVC 연결
* AOF 활성화
* Pod 재생성
* 기존 데이터 유지

### 판정

```text
Redis Persistence
→ PASS

Redis Authority-based DR
→ Partial
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

최종 E2E 측정 RTO는 **52초**이다.

---

# 7. Gate 요약

| Gate | 대상                       | 현재 상태       | 핵심 Evidence                              |
| ---- | ------------------------ | ----------- | ---------------------------------------- |
| A    | MariaDB / MaxScale       | `Validated` | DB Provider / Schema / Account           |
| B    | Credential / TLS         | `Validated` | Vault ↔ Secret ↔ DB                      |
| C    | NFS / Kubernetes Storage | `Validated` | PVC / Persistence                        |
| D    | MariaDB Backup           | `Validated` | Full / Incremental / Chain / Lock        |
| E    | MariaDB Recovery         | `Validated` | Restore / 정합성 / RTO / RPO                |
| F    | Redis Persistence        | `Validated` | Write / Read / Pod Recreation            |
| F    | Redis DR                 | `Partial`   | 권한 기반 Backup / Restore 미완료               |
| G    | etcd DR                  | `Validated` | Isolated Restore / Quorum / API / Object |

---

# 8. 남은 Blocker와 Gap

## 8.1 MariaDB Recovery 실측 결과 문서 반영

DR-01 실제 복구 실측은 완료되었으므로, 최종 문서에는 **실측 문서에서 확정된 RTO/RPO 값을 동일하게 반영해야 한다.**

초기 측정값은 최종 결과에서 제외한다.

```text
초기 측정값
→ 문서에서 제외

최종 DR-01 실측값
→ 공식 RTO / RPO로 사용
```

---

## 8.2 Redis 완전한 DR

현재 Redis는 다음까지 검증되어 있다.

```text
Write / Read
→ PVC
→ Pod Recreation
→ Data Persistence
```

하지만 별도의 Backup을 권한 있는 방식으로 보호하고 장애 상황에서 복구하는 전체 과정은 아직 완전히 검증되지 않았다.

따라서 현재 상태를 `Redis DR PASS`로 확대 해석하지 않고 `Partial`로 유지한다.

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

## 8.5 DB/MaxScale 서비스 레벨 Observability 보류

현재 Prometheus로 수집되는 것은 4대 서버의 자체 자원(node_exporter)뿐이다.
MariaDB/MaxScale **서비스 자체 상태**의 Prometheus 연동은 다음과 같이
보류된 상태다.

```text
mysqld_exporter
→ 배포·계정·Metric 수집 자체는 정상 동작
→ Prometheus 수집 연동은 확인 완료(2026-09-21), Alert Rule은 2차 이관

maxscale_exporter
→ 공식 exporter 부재(MXS-3022 Won't Do)로 소스 빌드 방식으로 2차 재착수 (REST read-only 계정만 코드화)
```

따라서 복제 지연이나 MaxScale Failover 발생을 Alert로 조기 탐지하는
경로는 아직 없으며, 장애 인지는 계속 수동 확인(`maxctrl list servers`,
로그 확인)에 의존한다. 이 Gap은 MariaDB DR Recovery 완료 기준(RTO/RPO)
과는 별개이며, 별도 팀 결정으로 관리한다.

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
MariaDB Recovery
    ↓
RTO / RPO Evidence
    ↓
Redis Persistence
    ↓
etcd DR
    ↓
Application Data Consumer
    ↓
MVP Data Platform Acceptance
```

### MariaDB Recovery 핵심 경로

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
RTO / RPO
```

이 흐름에서 하나라도 실패하면 단순히 Backup 파일이 존재한다는 이유로 Recovery 완료로 판단하지 않는다.

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
* [x] 최종 RTO / RPO 측정

> 최종 RTO / RPO 숫자는 DR-01 실측 결과 원문과 동일하게 반영한다.

---

## 10.4 Redis

* [x] Redis StatefulSet
* [x] PVC 연결
* [x] AOF Persistence
* [x] Write / Read
* [x] Pod Recreation 후 데이터 유지
* [ ] 권한 기반 Redis Backup / Restore DR

---

## 10.5 etcd DR

* [x] etcd Snapshot
* [x] Snapshot Hash / Revision
* [x] NFS 전달
* [x] 격리된 3-member Restore
* [x] Quorum 확인
* [x] kube-apiserver 인증
* [x] Kubernetes Object 비교
* [x] E2E RTO 측정

최종 측정 결과:

```text
etcd DR-02 RTO = 52초
```

---

# 11. Traceability

## 11.1 Database / Backup / Recovery

| 항목               | 추적 대상           | 목적                                    |
| ---------------- | --------------- | ------------------------------------- |
| DB 기본 구성         | `seokpan-infra` | MariaDB / MaxScale 구성                 |
| Backup Chain     | `seokpan-infra` | Full / Incremental Backup             |
| Backup 공유 상태     | `seokpan-infra` | NFS 기반 Chain 상태 공유                    |
| Backup 동시 실행 보호  | `seokpan-infra` | Shared Lock                           |
| MariaDB Recovery | `seokpan-infra` | Restore / Replication / Service Check |
| DR-01 실측         | Infra #194      | MariaDB RTO / RPO 실측                  |
| Recovery 버그 수정   | Infra PR #197   | Master / Replica Recovery 경로 오류 수정    |

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
| Recovery Safety   | Infra #176           | 잘못된 환경에서 DR 실행 방지                  |
| Replication       | Infra #197           | Master / Replica 조건별 Recovery 검증   |

---

## 11.4 Kubernetes DR

| 항목          | 추적 대상         | 목적                     |
| ----------- | ------------- | ---------------------- |
| etcd DR 설계  | Infra #113    | Snapshot / Restore     |
| etcd DR E2E | Infra PR #191 | 격리 환경 Restore 및 RTO 검증 |
| DR 보호 범위    | Infra #143 계열 | 전체 DR 범위 및 MVP 범위 관리   |

---

## 11.5 최종 판정

본 문서에서 중요한 판단 기준은 **구현 여부가 아니라 실제 검증 여부**이다.

```text
구성 완료
    ↓
실행
    ↓
검증
    ↓
측정
    ↓
Evidence
    ↓
Gate 판정
```

따라서 MariaDB Backup, Recovery, Redis Persistence, etcd DR은 각각 독립적으로 검증 상태를 관리하며, 하나의 구성요소가 정상이라고 해서 전체 Data Platform Recovery가 완료된 것으로 판단하지 않는다.

최종 MVP Data Platform의 완료 조건은 다음과 같다.

```text
MariaDB
+ MaxScale
+ Credential / TLS
+ NFS Storage
+ Backup
+ MariaDB Recovery
+ RTO / RPO Evidence
+ Redis Persistence
+ etcd DR
+ Application Data Consumer 검증
```

이 모든 항목의 실제 실행 결과와 Evidence를 기준으로 최종 상태를 판정한다.
