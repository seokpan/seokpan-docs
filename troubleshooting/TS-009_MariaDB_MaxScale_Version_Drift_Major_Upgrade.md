[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-009 — MariaDB/MaxScale 버전 불일치(Version Drift)와 주 버전 업그레이드(Major Upgrade) 후 시스템 테이블·Replication Channel 문제

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-31 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | 정태훈(애플리케이션 DB 연동), 이유빈(Ansible 재현성), 최유준(DB 모니터링·관측) |

## 최초 문제: 설계와 코드가 다른 버전

06 Ansible 자동화·테스트 설계 문서 기준과 실제 `main` 코드가 어긋나 있었다.

| 대상 | 설계 기준 | 실제 코드/초기 실제 실행 환경(Runtime) |
|---|---|---|
| MariaDB | `11.8.9 LTS` | `10.5.29` AppStream |
| MaxScale | `24.02.10` 목표 | `23.08.13` |

MariaDB 11.8.9는 CentOS Stream 9 AppStream에서 제공되지 않으므로 공식 MariaDB 패키지 저장소(Repository)로 설치 방식을 바꿔야 했다.

## 2차 역추적에서 확인한 실제 업그레이드 선행 장애 1: AppStream 잔존 패키지 의존성 충돌

실서버에 MariaDB 공식 패키지를 적용하는 과정에서 기존 AppStream 설치 시 함께 들어온 `mariadb-gssapi-server`가 구버전 `mariadb-server` 의존성을 유지해 `MariaDB-server-11.8.9` 전환이 막혔다.

핵심은 단순히 Package 이름을 바꾸는 것만으로는 Major Package Vendor 전환이 되지 않았다는 점이다. 프로젝트에서는 GSSAPI 인증을 사용하지 않으므로 `allowerasing`을 사용해 충돌 패키지를 함께 정리하도록 설치 경로를 보완했다.

```text
AppStream mariadb-server 10.5 + mariadb-gssapi-server
        ↓
MariaDB 공식 Repo 11.8.9 설치 시도
        ↓
기존 GSSAPI 패키지가 구버전 의존성 유지
        ↓
Depsolve 충돌
        ↓
미사용 GSSAPI 패키지 정리 + allowerasing
        ↓
공식 MariaDB 11.8.9 전환 진행
```

## 2차 역추적에서 확인한 자동화 결함: 패키지 저장소의 '존재'만 보고 목표 버전 Drift를 놓침

초기 `setup_repo` 멱등성 판정은 `mariadb-maxscale` 패키지 저장소가 **등록되어 있는지**만 확인했다. 그 결과 이미 `23.08` Repo가 등록된 서버에서는 목표가 `24.02`로 바뀌어도 “Repo 있음”으로 판정해 재설정을 건너뛸 수 있었다.

따라서 멱등성 기준을 다음처럼 바꿨다.

```text
잘못된 기준: 패키지 저장소가 존재하는가?
정확한 기준: 패키지 저장소가 존재하고, 그 저장소가 현재 목표 Version Series를 가리키는가?
```

MaxScale에서 먼저 수정한 뒤 MariaDB Repo 등록 로직에도 동일한 Version-Drift-aware 판정을 적용했다.

## MaxScale 24.02.10 → 24.02.9 실제 가용성 재확인

실서버에서 패키지 저장소를 확인하는 과정에서 한 번은 `24.02.10` 자체가 존재하지 않는 것으로 잘못 판단했고, 곧바로 공식 Release Note를 다시 대조해 **24.02.10 GA는 실제 존재**한다는 사실을 확인했다. 이후 `dnf clean all && dnf makecache`까지 수행해 다시 확인한 결과, 문제는 Release 존재 여부가 아니라 **프로젝트가 사용하는 Community 공개 Package 채널에서 24.02.10이 제공되지 않는 것**이었다.

즉 2차 역추적 결과 이 사건의 정확한 판단 변화는 다음과 같다.

```text
실서버 조회: 24.02.9까지만 보임
        ↓
중간 오판: 24.02.10은 존재하지 않는 버전인가?
        ↓
공식 Release Note 재검증: 24.02.10 GA는 실제 존재
        ↓
패키지 저장소 Cache/배포 채널 재검증
        ↓
최종 결론: GA는 존재하지만 Community 공개 Repo Package는 24.02.9까지만 제공
```

따라서 `24.02.9` 채택은 “24.02.10이 없는 버전이라서”가 아니라 **현재 프로젝트가 재현 가능한 무료 설치 경로에서 실제 확보 가능한 최신 24.02 패치가 24.02.9였기 때문**이다.

따라서 실제 배포 가능한 버전과 기능 호환성을 확인해 **24.02.9**로 구현 기준을 조정했다.

이것은 임의 Downgrade가 아니라:

```text
문서 목표 버전
    ↓
실제 공식 배포 채널 재검증
    ↓
사용 가능 Package 확인
    ↓
프로젝트 사용 기능 호환성 확인
    ↓
24.02.9로 구현 Freeze
```

과정이었다.

## Major Upgrade 후 실제 장애 1: 시스템 테이블 불일치

MariaDB `10.5.29 → 11.8.9` 패키지 교체 후:

```text
[ERROR] Incorrect definition of table mysql.event ...
[ERROR] mariadbd: Event Scheduler: An error occurred when initializing system tables.
```

이 발생했다.

## 원인 분석

패키지는 11.8.9로 바뀌었지만 `mysql.event` 등 시스템 테이블이 10.5 구조로 남아 있었다.

## 조치

양쪽 MariaDB에서:

```bash
mariadb-upgrade -u root -p
```

실행 후 재기동.

## 검증

Event Scheduler 정상화, 기존 오류 소거.

---

## Major Upgrade 후 실제 장애 2: MaxScale은 Running인데 Replication Channel이 없음

업그레이드 후:

```text
maxctrl list servers
```

에서는 두 서버 모두 Running으로 보였지만, `mariadb-02`에서:

```sql
SHOW SLAVE STATUS\G
```

를 실행하면:

```text
Empty set
```

이었다.

## 핵심 교훈

> **MaxScale이 Server를 Running으로 보는 것과 MariaDB Replication Channel이 실제 구성되어 있는 것은 동일한 검증이 아니다.**

## 원인 분석

- `gtid_slave_pos`는 보존
- 실제 Replication Connection 정보는 사라진 것으로 추정
- 데이터 자체는 정상

## 데이터 무결성 확인

- Binary Log / GTID / repl_user 확인
- `stone_game` 7개 테이블 Master/Replica 전량 비교
- 데이터 유실 없음

## 복구

기존 GTID Position을 이용해:

```sql
CHANGE MASTER TO
    MASTER_HOST='192.168.52.40',
    MASTER_PORT=3306,
    MASTER_USER='repl_user',
    MASTER_PASSWORD='<secret>',
    MASTER_USE_GTID=slave_pos;

START SLAVE;
```

## 최종 검증

- `Slave_IO_Running: Yes`
- `Slave_SQL_Running: Yes`
- `Seconds_Behind_Master: 0`
- IO/SQL Error 없음

## Before → After

```text
Before
"서비스가 Running이면 Replication도 정상일 것"
        ↓
Major Upgrade
        ↓
MaxScale Running / SHOW SLAVE STATUS = Empty set

After
서비스 상태와 Replication 상태를 분리 검증
        ↓
GTID 기반 Channel 재연결
        ↓
IO/SQL Thread + Lag + Error까지 완료조건에 포함
```

## 관련 근거

- Issue #63: https://github.com/seokpan/seokpan-infra/issues/63
- PR #65: https://github.com/seokpan/seokpan-infra/pull/65
- 구현 버전 기록 Docs PR #10: https://github.com/seokpan/seokpan-docs/pull/10