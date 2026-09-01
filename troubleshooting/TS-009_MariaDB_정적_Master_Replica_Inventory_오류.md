[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-009 — Auto Failover 환경에서 정적 `mariadb_master` / `mariadb_replica` Inventory가 잘못된 모델이 됨

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-31 |
| **상태** | **해결** |
| **주 담당** | **이유빈(Inventory) + 김상희(DB Automation)** |
| **영향 범위** | 정태훈(애플리케이션의 DB 사용 경로) |

## 문제 개요

초기 Inventory는 다음처럼 서버 이름과 DB 역할을 고정해서 표현했다.

```text
mariadb_master
└─ mariadb-01

mariadb_replica
└─ mariadb-02
```

그러나 Backup 검증과 MariaDB 업그레이드 과정에서 MaxScale Auto Failover로 실제 Master/Replica 역할이 바뀌는 것이 확인됐다.

## 원인 분석

Ansible Inventory는 서버의 정체성을 표현해야 하는데, 바뀔 수 있는 실제 DB 역할까지 정적으로 고정하고 있었다.

```text
mariadb-01 = 서버 정체성
Primary = 실행 시점의 가변 상태

두 개념은 동일하지 않음
```

이 구조를 유지하면 Failover 이후 자동화가 실제 역할과 다른 서버를 Master로 취급할 수 있었다.

## 1차 조치

Inventory를 다음처럼 서버 집합 중심으로 변경했다.

```text
database
├─ mariadbs
│  ├─ mariadb-01
│  └─ mariadb-02
└─ maxscale
   └─ maxscale-01
```

## 후속으로 확인된 코드 불일치

Inventory 구조를 바꾼 뒤에도 `playbooks/mariadb.yml`이 기존 그룹 이름을 계속 참조하고 있었다.

```text
hosts: mariadb_master:mariadb_replica
```

즉 Inventory만 수정하고 이를 사용하는 Playbook의 대상 그룹을 함께 바꾸지 않은 상태였다.

## 2차 조치

- `hosts: mariadbs`로 변경
- MariaDB/MaxScale/NFS Role과 Playbook 전체에서 옛 그룹 이름 검색
- `mariadb_master` / `mariadb_replica` 참조 제거 확인

## 검증

```text
mariadb-01 → 정상 대상
mariadb-02 → 정상 대상
unreachable=0
failed=0
```

MaxScale Playbook도 새 Inventory 구조에서 추가 수정 없이 정상 동작하는 것을 확인했다.

## 운영에 반영한 내용

- Inventory는 서버 정체성 중심으로 관리
- 실제 Master/Replica 역할은 `maxctrl list servers` 등 실행 시점 정보로 확인
- 애플리케이션은 특정 MariaDB Host가 아니라 VIP/MaxScale 경로를 사용

## 관련 근거

- 초기 정적 그룹 PR #33: https://github.com/seokpan/seokpan-infra/pull/33
- Inventory 재구조화 PR #70: https://github.com/seokpan/seokpan-infra/pull/70
- Follow-up Issue #71: https://github.com/seokpan/seokpan-infra/issues/71
- Playbook 수정 PR #72: https://github.com/seokpan/seokpan-infra/pull/72
