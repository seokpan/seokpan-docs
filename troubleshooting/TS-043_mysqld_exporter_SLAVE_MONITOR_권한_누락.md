[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-043 — mysqld_exporter slave_status collector의 SLAVE MONITOR 권한 누락으로 Access denied 발생

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-16 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | mysqld_exporter Runtime 검증(mariadb-01/02), 운영 서버 데이터 영향 없음 |

## 최초 문제

이슈 #199(Observability Exporter 코드화) 작업 중 PR #201에서
mysqld_exporter를 mariadb-01/02에 배포하고 `/metrics` 정상 응답을
확인했으나, `slave_status` collector가 아래 에러를 반환했다.

Error 1227 (42000): Access denied; you need (at least one of) the SLAVE MONITOR privilege(s) for this operation

## 원인

exporter 전용 계정 `exporter_svc`에는 mysqld_exporter 공식 문서 기준
권한인 `PROCESS`, `SELECT`, `BINLOG MONITOR`만 부여되어 있었다. 그러나
실제 실행 환경인 MariaDB 11.8.9에서는 Replica 상태(`SHOW SLAVE STATUS`
기반) 조회에 필요한 `SLAVE MONITOR` 권한이 별도로 요구되며, 이 권한이
없으면 `slave_status` collector가 위 에러로 즉시 실패한다.

## 조치

`exporter_svc` 계정 GRANT에 `SLAVE MONITOR` 권한을 추가했다.

## 검증

- 수정 전: db1/db2 모두 mysqld_exporter journal에서 위 Access denied
  에러 반복 확인
- 수정 후 재검증(2026-09-16, 실서버):
  - `mysql_exporter_collector_success{collector="collect.slave_status"} 1`
    — db1/db2 모두 확인
  - `mysql_up 1` — db1/db2 모두 확인
  - db2(Replica) 기준 `mysql_slave_status_slave_io_running 1`,
    `mysql_slave_status_slave_sql_running 1`,
    `mysql_slave_status_seconds_behind_master 0`,
    `mysql_slave_status_last_io_errno 0`,
    `mysql_slave_status_last_sql_errno 0` 확인
  - 최근 exporter journal에서 SLAVE MONITOR 관련 에러 재발 없음 확인

## Before → After

Before
exporter_svc: PROCESS, SELECT, BINLOG MONITOR
+ slave_status collector 실행 시 Access denied(1227)로 실패

After
exporter_svc: PROCESS, SELECT, BINLOG MONITOR, SLAVE MONITOR
+ slave_status collector 정상 수집, Replica 상태 지표 노출

## 관련 근거

- Issue #199: https://github.com/seokpan/seokpan-infra/issues/199
- PR #201: https://github.com/seokpan/seokpan-infra/pull/201
