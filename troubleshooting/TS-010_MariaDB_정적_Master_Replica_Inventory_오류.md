[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-010 — Auto Failover 환경에서 정적 `mariadb_master` / `mariadb_replica` Inventory가 잘못된 모델이 됨

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-31 |
| **상태** | **해결** |
| **주 담당** | **이유빈(Inventory) + 김상희(DB Automation)** |
| **영향 범위** | 정태훈(애플리케이션의 DB 사용 경로) |

## 문제 개요

초기 Inventory에는:

```text
mariadb_master:
  hosts:
    mariadb-02

mariadb_replica:
  hosts:
    mariadb-01
```

가 있었다.

그러나 실제 DB는 MaxScale 자동 장애조치(Auto Failover)를 사용한다.

## 문제

Master/Replica 역할이 바뀔 수 있는데 Inventory가 이를 고정했다.

결국 Ansible이 현재 역할을 잘못 추론할 가능성이 있었다.

## 1차 조치

```yaml
database:
  hosts:
    mariadb-01:
    mariadb-02:
```

로 통합했다.

## 후속 문제

Inventory만 바꿨다고 끝난 게 아니었다.

Playbook에:

```yaml
hosts: mariadb_master
```

가 남아 있으면 Inventory 개선을 소비하지 못한다.

실제로 `mariadb_backup.yml`에서 이 참조가 발견됐다.

## 2차 조치

Backup Playbook도:

```text
mariadb_master
→ database
```

로 변경했다.

그리고 Role 내부에서 MaxScale을 통해 **실제 실행 환경(Runtime) Master를 판별**하도록 정리했다.

## 검증

```text
Inventory parse PASS

mariadb-01:
Replication role 확인

mariadb-02:
Replication role 확인
```

## 담당 역할 및 영향

- 이유빈: Inventory 구조
- 김상희: Backup Role/DB Runtime 판별
- 정태훈: Application은 개별 DB Host가 아니라 VIP/MaxScale만 사용

## 관련 근거

- Issue #58: https://github.com/seokpan/seokpan-infra/issues/58
- PR #61: https://github.com/seokpan/seokpan-infra/pull/61
- Backup 후속 Fix PR #65: https://github.com/seokpan/seokpan-infra/pull/65
