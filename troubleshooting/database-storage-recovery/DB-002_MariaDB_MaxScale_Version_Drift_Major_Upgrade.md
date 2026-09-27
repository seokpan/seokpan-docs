[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-008 — MariaDB/MaxScale 버전 불일치(Version Drift)와 주 버전 업그레이드(Major Upgrade) 후 시스템 테이블·Replication Channel 문제

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-31 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | 정태훈(애플리케이션 DB 연동), 이유빈(Ansible 재현성), 최유준(DB 모니터링·관측) |

## 최초 문제: 설계와 코드의 버전 불일치

06 Ansible 자동화·테스트 설계 문서 기준과 실제 `main` 코드가 어긋나 있었다.

| 대상 | 설계 기준 | 실제 코드/초기 실행환경 |
|---|---|---|
| MariaDB | `11.8.9 LTS` | `10.5.29` AppStream |
| MaxScale | `24.02.10` 목표 | `23.08.13` |

MariaDB 11.8.9는 CentOS Stream 9 AppStream에서 제공되지 않으므로 MariaDB 공식 패키지 저장소(Repository)를 사용하는 방식으로 변경해야 했다.

## 추가 검증에서 확인한 업그레이드 선행 장애: AppStream 잔존 패키지 의존성 충돌

MariaDB 공식 패키지를 적용하는 과정에서 기존 AppStream 설치 시 함께 들어온 `mariadb-gssapi-server`가 구버전 `mariadb-server` 의존성을 유지해 `MariaDB-server-11.8.9` 전환이 막혔다.

프로젝트에서는 GSSAPI 인증을 사용하지 않으므로 `allowerasing`을 이용해 충돌 패키지를 함께 정리하도록 설치 경로를 보완했다.

```text
AppStream mariadb-server 10.5 + mariadb-gssapi-server
→ MariaDB 공식 Repository 11.8.9 설치 시도
→ 기존 GSSAPI 패키지의 구버전 의존성 충돌
→ 미사용 GSSAPI 패키지 정리 + allowerasing
→ 공식 MariaDB 11.8.9 전환
```

## 패키지 저장소 존재 여부만 확인하던 자동화 결함

초기 `setup_repo` 멱등성 판정은 `mariadb-maxscale` 저장소가 등록되어 있는지만 확인했다. 이미 `23.08` 저장소가 등록된 서버에서는 목표가 `24.02`로 바뀌어도 “저장소 있음”으로 판단해 재설정을 건너뛸 수 있었다.

따라서 기준을 다음처럼 바꿨다.

```text
잘못된 기준: 저장소가 존재하는가?
정확한 기준: 저장소가 존재하고 현재 목표 Version Series를 가리키는가?
```

MaxScale에서 먼저 수정한 뒤 MariaDB 저장소 등록 로직에도 같은 버전 불일치 감지 기준을 적용했다.

## MaxScale 24.02.10 → 24.02.9 가용성 재확인

실서버에서 패키지 저장소를 확인하는 과정에서 `24.02.10` 자체가 없는 버전으로 한 차례 잘못 판단했다. 공식 Release Note를 다시 확인한 결과 **24.02.10 GA는 실제 존재**했다.

이후 패키지 저장소 Cache와 배포 채널을 다시 확인해, 문제는 Release 존재 여부가 아니라 **프로젝트가 사용하는 Community 공개 패키지 채널에서 24.02.10 패키지를 제공하지 않는 것**임을 확인했다.

따라서 현재 프로젝트에서 재현 가능한 무료 설치 경로에서 확보 가능한 최신 24.02 패치인 **24.02.9**를 구현 기준으로 확정했다.

## Major Upgrade 후 장애 1: 시스템 테이블 불일치

MariaDB `10.5.29 → 11.8.9` 패키지 교체 후 다음 오류가 발생했다.

```text
[ERROR] Incorrect definition of table mysql.event ...
[ERROR] mariadbd: Event Scheduler: An error occurred when initializing system tables.
```

패키지는 11.8.9로 바뀌었지만 `mysql.event` 등 시스템 테이블이 10.5 구조로 남아 있었다.

### 조치

양쪽 MariaDB에서 다음을 실행한 뒤 재기동했다.

```bash
mariadb-upgrade -u root -p
```

### 검증

Event Scheduler가 정상화되고 기존 시스템 테이블 오류가 사라진 것을 확인했다.

## Major Upgrade 후 장애 2: MaxScale은 Running인데 Replication Channel이 없음

업그레이드 후 `maxctrl list servers`에서는 두 서버 모두 Running으로 보였지만 `mariadb-02`의 `SHOW SLAVE STATUS\G` 결과는 `Empty set`이었다.

즉 **MaxScale이 서버를 Running으로 판단하는 것과 MariaDB Replication Channel이 실제 구성되어 있는 것은 별개의 검증 대상**이었다.

### 데이터 무결성 확인

- Binary Log / GTID / `repl_user` 확인
- `stone_game` 7개 테이블 Master/Replica 전량 비교
- 데이터 유실 없음

### 복구

기존 GTID Position을 사용해 Replication Channel을 다시 연결했다.

```sql
CHANGE MASTER TO
    MASTER_HOST='192.168.52.40',
    MASTER_PORT=3306,
    MASTER_USER='repl_user',
    MASTER_PASSWORD='<secret>',
    MASTER_USE_GTID=slave_pos;
START SLAVE;
```

### 최종 검증

- `Slave_IO_Running: Yes`
- `Slave_SQL_Running: Yes`
- `Seconds_Behind_Master: 0`
- IO/SQL Error 없음

## Before → After

```text
Before
서비스가 Running이면 Replication도 정상이라고 판단

After
서비스 상태와 Replication 상태를 분리 검증
→ GTID 기반 Channel 재연결
→ IO/SQL Thread + Lag + Error까지 완료조건에 포함
```

## 관련 근거

- Issue #63: https://github.com/seokpan/seokpan-infra/issues/63
- PR #65: https://github.com/seokpan/seokpan-infra/pull/65
- 구현 버전 기록 Docs PR #10: https://github.com/seokpan/seokpan-docs/pull/10
