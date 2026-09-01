[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-010 — Auto Failover 환경에서 정적 `mariadb_master` / `mariadb_replica` Inventory가 잘못된 모델이 됨

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 아래 내용만으로 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성했습니다.

| 항목 | 내용 |
|---|---|
| **시점** | 2026-08-31 |
| **상태** | **해결** |
| **주 담당** | **이유빈(Inventory) + 김상희(DB Automation)** |
| **영향 역할** | 정태훈(애플리케이션의 DB 사용 경로) |
| **핵심 범주** | Ansible Inventory / Dynamic Runtime Role |

## 배경

초기 Inventory는:

```text
mariadb_master
└─ mariadb-01

mariadb_replica
└─ mariadb-02
```

처럼 서버 이름과 런타임 역할을 동일시했다.

그러나 TS-004, TS-009 검증에서 Auto Failover로 역할이 실제로 바뀌는 것이 확인됐다.

## 문제

Ansible Inventory는 Host Identity를 표현해야 하는데, 바뀔 수 있는 Runtime Role을 정적으로 박아두고 있었다.

```text
mariadb-01 = Host Identity
Primary = Runtime State

둘은 같은 개념이 아님
```

## 1차 조치

Inventory를:

```text
database
├─ mariadbs
│  ├─ mariadb-01
│  └─ mariadb-02
└─ maxscale
   └─ maxscale-01
```

로 변경했다.

## 후속 문제

Inventory 변경 자체는 정상 반영됐지만, `playbooks/mariadb.yml`이 여전히:

```text
hosts: mariadb_master:mariadb_replica
```

를 참조하고 있었다.

즉 **Inventory 구조는 바뀌었지만 이를 사용하는 Playbook 코드가 옛 그룹 이름을 계속 사용**하고 있었다.

새 Inventory에서 Playbook 대상 Host를 찾지 못할 수 있는 상태였다.

## 2차 조치

- `hosts: mariadbs`로 변경
- MariaDB/MaxScale/NFS Role과 Playbook 전체 검색
- 옛 `mariadb_master` / `mariadb_replica` 참조 제거 확인

## 검증

```text
mariadb-01 → 정상 대상
mariadb-02 → 정상 대상
unreachable=0
failed=0
```

MaxScale Playbook도 추가 수정 불필요 확인.

## 역할 영향

- **이유빈:** Inventory 구조 제공 방식 변경
- **김상희:** 새 Inventory를 소비하는 MariaDB Playbook 후속 정합화
- **정태훈:** Application은 최종적으로 고정 DB Host가 아니라 VIP/MaxScale 경로를 소비

## 근거

- 초기 정적 그룹 PR #33: https://github.com/seokpan/seokpan-infra/pull/33
- Inventory 재구조화 PR #70: https://github.com/seokpan/seokpan-infra/pull/70
- Follow-up Issue #71: https://github.com/seokpan/seokpan-infra/issues/71
- Playbook 수정 PR #72: https://github.com/seokpan/seokpan-infra/pull/72

---
