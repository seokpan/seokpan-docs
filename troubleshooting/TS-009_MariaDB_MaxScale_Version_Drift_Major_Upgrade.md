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

검토 당시:

```text
MariaDB 문서 기준: 11.8 계열
코드: 10.5

MaxScale 코드:
MariaDB repo 10.5
MaxScale repo 23.08
```

즉 실제 설치·실행 상태(Runtime)가 프로젝트 문서와 달랐다.

## 2차 역추적에서 확인한 실제 업그레이드 선행 장애 1: AppStream 잔존 패키지 의존성 충돌

MariaDB를 11.8.9로 전환하는 과정에서 단순 Repository 변경만으로 끝나지 않았다.

기존 CentOS AppStream 계열 패키지인:

```text
mariadb-gssapi-server-3:10.5.29-1.el9.x86_64
```

가 다음 의존성을 요구했다.

```text
mariadb-server = 3:10.5.29-1.el9
```

반면 목표 MariaDB는 `11.8.9`였기 때문에 DNF depsolve 단계에서 충돌했다.

즉 Major Upgrade를 막은 첫 실제 장애는 **버전 문자열만 바꾸지 않았기 때문**이 아니라, 구 AppStream 패키지의 잔존 의존성이었다.

후속 조치에서는 이 충돌 패키지를 식별하고 목표 MariaDB 계열과 충돌하지 않도록 제거·정리한 뒤 설치를 진행했다.

## 2차 역추적에서 확인한 자동화 결함: Repository의 '존재'만 보고 목표 버전 Drift를 놓침

MaxScale Repository 작업에서는 또 다른 멱등성 문제가 드러났다.

기존 구현은 `mariadb.repo` 파일 또는 MariaDB Repository가 이미 존재하면 "Repo 준비 완료"로 간주할 수 있었다. 하지만 당시 파일 안에는:

```text
MaxScale/23.08
```

이 남아 있었다.

목표는 `24.02` 계열이므로 실제로는 **Repo가 존재하는 것과 올바른 Repo가 구성되어 있는 것은 다른 조건**이었다.

따라서 자동화 검증도:

```text
파일 존재?
```

가 아니라 최소한:

```text
MariaDB 11.8 Repo인가?
MaxScale 24.02 Repo인가?
```

까지 확인하도록 바뀌어야 했다.

이 문제는 새로 설치하는 경우보다 기존 서버를 재사용할 때 더 위험하다. 잘못된 Repository가 이미 있으면 Ansible이 "이미 구성됨"으로 보고 지나갈 수 있기 때문이다.

## MaxScale 24.02.10 → 24.02.9 실제 가용성 재확인

버전 조정 과정에서는 `24.02.10`을 둘러싼 판단도 한 번 수정됐다.

초기 확인에서는 실제 Repository Metadata에서 패키지가 보이지 않았고, 그 결과 `24.02.10` 자체가 존재하지 않는 버전처럼 판단할 가능성이 있었다.

그러나 공식 Release Note를 다시 확인하자 **MaxScale 24.02.10 GA Release 자체는 존재**했다.

즉 질문을 다음처럼 분리해야 했다.

```text
24.02.10 Release가 존재하는가?
→ Yes

현재 Community 공개 Repository에서
우리가 설치할 수 있는 패키지가 제공되는가?
→ 검증 시점에는 확인되지 않음
```

프로젝트가 실제 패키지로 재현 가능한 버전을 우선했기 때문에 최종적으로 `24.02.9`를 사용했다.

이 기록은 "24.02.10이 없는 버전"이라고 쓰지 않는다. **Release 존재 여부와 현재 설치 가능한 공개 패키지 가용성을 분리해서 판단한 사례**다.

## Major Upgrade 후 실제 장애 1: 시스템 테이블 불일치

업그레이드 후 DB 시작 로그에:

```text
Column count of mysql.proc is wrong
```

오류가 나타났다.

## 원인 분석

MariaDB Binary는 11.8로 올라갔지만 `/var/lib/mysql`의 System Table은 10.5 시절 구조였다.

## 조치

```bash
mariadb-upgrade --force
```

후 서비스 재시작.

## 검증

오류 제거 확인.

---

## Major Upgrade 후 실제 장애 2: MaxScale은 Running인데 Replication Channel이 없음

겉으로는:

```text
maxscale.service: active
mariadb.service: active
```

였다.

그런데:

```sql
SHOW SLAVE STATUS\G
```

결과가:

```text
Empty set
```

이었다.

## 핵심 교훈

```text
Service Running
≠
Replication 정상
```

## 원인 분석

`mariadb-01`은 실제 Replica였지만 복제 Channel 자체가 없었다.

따라서 MaxScale이 Running이어도 DB HA가 정상이라고 볼 수 없었다.

## 데이터 무결성 확인

먼저 7개 테이블의 건수를 비교했다.

결과:

- 양쪽 동일
- 유실 없음

## 복구

실제 Master 확인 후 Replica에:

```sql
CHANGE MASTER TO ...;
START SLAVE;
```

으로 복제 Channel을 재구성했다.

## 최종 검증

```text
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
Seconds_Behind_Master: 0
```

MaxScale:

```text
mariadb-02 → Master
mariadb-01 → Slave, Running
```

## Before → After

```text
Before
Service active만 확인
        ↓
Replication Channel 없음
        ↓
HA가 실제로는 불완전

After
System Table Upgrade
        ↓
Replication Channel 복구
        ↓
MaxScale Role + SHOW SLAVE STATUS 교차검증
```

## 관련 근거

- Version Lock Parent #38: https://github.com/seokpan/seokpan-infra/issues/38
- DB Version Issue #39: https://github.com/seokpan/seokpan-infra/issues/39
- 실제 작업 Issue #43: https://github.com/seokpan/seokpan-infra/issues/43
- PR #48: https://github.com/seokpan/seokpan-infra/pull/48
