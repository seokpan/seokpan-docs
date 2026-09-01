[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-003 — MariaDB Backup 검증 중 실제 Master 역전과 DCL(계정·권한 변경 SQL) 복제 충돌로 SQL Thread 정지

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-28 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | 정태훈(애플리케이션의 DB 사용 경로), 이유빈(Inventory), DB를 사용하는 전체 구성요소 |

## 문제 개요

`backup_svc` 계정을 만들고 MariaDB Backup/Restore를 검증하던 중 Replica 반영 여부를 확인하는 과정에서 `mariadb-01`의 SQL Thread가 정지한 상태가 발견됐다.

핵심 오류는 다음과 같았다.

```text
Last_SQL_Errno: 1396
Last_SQL_Error: Error 'Operation DROP USER failed for 'app_user'@'%'' on query
```

## 발생 경위

1. `mariadb-01`이 Master였던 시점에 계정 생성·권한 부여와 `app_user` 정리 작업이 수행됐다.
2. 이후 장애조치(Auto Failover)로 `mariadb-02`가 새 Master로 승격됐다.
3. 새 Master에서 동일한 계정 정리 작업이 다시 수행됐다.
4. 해당 변경이 `mariadb-01`로 복제되는 과정에서 이미 존재하지 않는 `app_user`에 대한 `DROP USER`가 다시 실행됐다.
5. 그 결과 Replica의 SQL Thread가 오류 1396에서 정지했다.

## 원인 분석

문서와 초기 Inventory에서 특정 서버를 Primary로 고정해 생각한 것과 실제 실행 시점의 Master 역할이 달랐다. 또한 Failover 전·후 양쪽 Master에서 동일한 계정 관련 DCL이 수행되면서 복제 충돌이 발생했다.

`slave_ddl_exec_mode=IDEMPOTENT` 설정이 있더라도 `CREATE USER`, `DROP USER`, `GRANT` 같은 계정·권한 변경 SQL까지 자동으로 안전하게 처리해주는 것은 아니었다.

## 데이터 유실 여부 확인

- 실제 신규 Master의 `identity_svc`, `game_svc` 계정과 GRANT 정상
- 서비스 스키마 데이터의 실질적 유실 없음
- 문제는 Replica SQL Thread가 DCL 한 건에서 정지한 것이었음

## 즉시 복구

```sql
STOP SLAVE SQL_THREAD;
SET GLOBAL sql_slave_skip_counter = 1;
START SLAVE SQL_THREAD;
```

복구 후 현재 Master 계열의 GTID가 양쪽에서 일치하는지 확인했다.

## 운영 기준으로 반영한 내용

DB에 쓰기 작업을 하기 전에 문서나 호스트 이름으로 Master를 추정하지 않고 다음 명령으로 실제 Master를 확인하도록 했다.

```text
maxctrl list servers
```

이 사건을 계기로 이후 Inventory도 서버 정체성과 가변적인 DB 역할을 분리하는 방향으로 정리했고, GTID 및 자동 장애조치 설정 역시 실제 역할 변경을 전제로 다시 검토했다.

## DR 검증으로 이어진 결과

같은 날 수행한 DR-01 검증에서는 다음을 확인했다.

- 실제 Master: `mariadb-02`
- Full Backup → Prepare → SHA-256 → NFS Staging
- 격리 데이터 디렉터리 + 3307 포트 Restore
- `member 2`, `game 2`, `move 9` Count 일치
- **RTO: 3분 23초**
- 테스트 시점 데이터 손실: 0건
- 정기 Backup Schedule이 아직 없으므로 RPO는 최대 Backup 주기만큼으로 정의

## Before → After

```text
Before
서버 이름을 기준으로 Master 역할을 고정해서 판단
→ Failover 이후에도 같은 전제로 DCL 수행
→ 중복 DCL이 Replica에서 충돌
→ SQL Thread 정지

After
MaxScale에서 실제 Master 확인
→ 실제 역할 기준으로 작업
→ Replication 상태 확인을 완료조건에 포함
→ Backup/Restore와 RTO/RPO 실측
```

## 관련 근거

- Backup Issue #43: https://github.com/seokpan/seokpan-infra/issues/43
- DR-01 Issue #45: https://github.com/seokpan/seokpan-infra/issues/45
