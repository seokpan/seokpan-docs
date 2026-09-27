[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-042 — MariaDB DR 복구 replication_setup의 role Guard 누락으로 master 경로 fatal 실패

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-16 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | MariaDB DR-01 운영 재개 자동화(`mariadb_dr_recovery.yml`), 운영 서버 데이터 영향 없음 |

## 최초 문제

이슈 #194 DR-01 실측(mariadb-01/02 양쪽 동시 유실 시나리오 재현) 중,
`mariadb_dr_recovery.yml --tags replication -e backup_restore_role=master`
실행이 아래 에러로 fatal 처리됐다.

```
Error while resolving value for 'fail_msg': object of type 'dict' has no attribute 'query_result'
```

## 원인

`replication_setup.yml`의 `CHANGE MASTER TO` / `START SLAVE` /
`SHOW SLAVE STATUS` 3개 태스크는 모두 `when: backup_restore_role ==
'replica'` 조건이 걸려 있어 master 경로에서는 스킵되고, `dr_slave_status`
변수 자체가 register되지 않는다. 그런데 바로 다음의 "복제 정상 기동
Gate"(`ansible.builtin.assert`) 태스크에만 이 조건이 빠져 있어,
master 경로에서도 항상 실행을 시도하며 존재하지 않는
`dr_slave_status.query_result`를 참조하다 실패했다.

## 조치

"복제 정상 기동 Gate" 태스크에 같은 파일의 다른 3개 태스크와 동일한
`when: backup_restore_role == 'replica'` 조건을 추가했다(PR #197).

## 검증

- 수정 전: mariadb-01(role=master) 실행에서 위 에러로 fatal 재현(이슈
  #194 코멘트)
- 수정 후 재검증(2026-09-16, 실서버):
  - mariadb-01(role=master): "복제 정상 기동 Gate" 포함 replica 전용
    태스크 전부 `skipping` 처리, `failed=0`으로 Play 정상 완주
  - mariadb-02(role=replica): `SHOW SLAVE STATUS` 및 "복제 정상 기동
    Gate" 정상 실행, `Slave_IO_Running=Yes`/`Slave_SQL_Running=Yes`
    확인, `failed=0`으로 기존 정상 동작 유지 확인(회귀 없음)

## Before → After

```
Before
replica 전용 Gate 태스크에 role 조건 누락
+ master 경로 실행 시 미정의 변수 참조로 fatal

After
Gate 태스크에도 동일 when 조건 적용
+ master/replica 양쪽 경로 모두 정상 완주
```

## 관련 근거

- Issue #194: https://github.com/seokpan/seokpan-infra/issues/194
- PR #197: https://github.com/seokpan/seokpan-infra/pull/197
