[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-017 — MariaDB `read_only` 정적 설정이 자동 장애조치(Auto Failover)와 충돌하고 GTID(복제 트랜잭션 식별자) Domain 설정도 잘못 분리됨

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-01 |
| **상태** | **핵심 해결 · Manual Switchover 후속 있음** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | 정태훈(애플리케이션 DB 연동), 리뷰를 통한 플랫폼 검증 |

## 최초 문제

`mariadb-02`의 `custom.cnf`에:

```text
read_only=ON
```

이 정적으로 들어 있었다.

자동 장애조치(Auto Failover)로 이 서버가 Master가 되면 Runtime에서는 MaxScale이 read_only를 OFF로 바꿀 수 있다.

하지만 재부팅하면 설정 파일이 다시 적용되어:

```text
Master인데 read_only=ON
```

이 될 수 있다.

## 1차 조치

정적 Config에서 `read_only` 제거.

그리고 MaxScale:

```ini
enforce_read_only_slaves=true
```

로 역할 기반 Runtime 관리로 전환했다.

## 실제 Failover 검증

단순 설정 비교가 아니라 실제 Master Reboot를 수행했다.

### 첫 번째 강제 Master Reboot

Failover Threshold를 넘겨 실제 Auto Failover 발생.

```text
약 13초 후 mariadb-01 승격
새 Master read_only=OFF
```

### 두 번째 Reboot

Threshold 이내에 자력 복구해 Failover 없음.

```text
기존 Master 유지
read_only=OFF
```

### Replica

```text
read_only=ON
```

정상 수렴.

## 추가 발견: Auto Rejoin이 GTID Divergence에서 거부됨

첫 강제 Failover에서 구 Master와 새 Master의 GTID가 갈라졌다.

MaxScale은 이를 안전하지 않은 상태로 보고 Auto Rejoin을 거부했다.

이는 장애가 아니라 안전장치였다.

수동 `mariadb-backup` 재구축으로 다시 편입했다.

두 번째 Test에서는 GTID가 맞아 Auto Rejoin 성공.

## 리뷰에서 추가 발견: 서버별 `gtid_domain_id=1/2`는 현재 Topology와 맞지 않음

초기 PR에는:

```text
mariadb-01 → gtid_domain_id=1
mariadb-02 → gtid_domain_id=2
```

가 있었다.

1 Primary + 1 Replica에서 양쪽이 자동으로 역할을 교환하는 구조라면 두 서버가 같은 Replication Domain을 공유해야 한다.

따라서:

```text
server_id
→ 서로 다름

gtid_domain_id
→ 둘 다 1
```

로 수정했다.

이 부분은 단순 검토 코멘트로 끝내지 않고 실제 PR 상태를 재확인했다.

담당자가 수정 결과를 댓글로 제시한 시점에도 실제 Pull Request의 최신 코드(Head)에는 한동안:

```text
mariadb-02 gtid_domain_id: 2
```

가 남아 있었다.

즉:

```text
"수정했다"는 설명
≠
실제 Pull Request 최신 코드에 수정 Commit 존재
```

였다.

후속 Commit이 실제로 Push된 뒤 최신 코드를 다시 확인해:

```text
mariadb-01: gtid_domain_id: 1
mariadb-02: gtid_domain_id: 1
```

을 확인했다.

## 최종 수정

```text
mariadb-01
server_id=1
gtid_domain_id=1

mariadb-02
server_id=2
gtid_domain_id=1
```

## 재검증

- Replica IO/SQL `Yes`
- 새 Write는 Domain 1
- 과거 Domain 2 이력은 더 증가하지 않음
- MaxScale:
  - Master
  - Slave
- read_only:
  - Master OFF
  - Slave ON
- Pull Request 최신 코드에서 양쪽 `gtid_domain_id: 1` 확인
- 최종 검토 승인
- PR #88 `main` 브랜치 Merge 완료

## 남은 문제: Manual Switchover Timeout

자동 Failover는 동작하지만 수동 Switchover Command Timeout은 별도 문제로 남아 있다.

PR #88 자체를 막는 문제는 아니지만 **Recovery/운영 검증 전에 별도 Issue로 추적해야 하는 항목**이다.

## Before → After

```text
Before
파일이 역할을 고정
+ GTID Domain도 서버별 분리

After
Runtime 역할은 MaxScale이 관리
+ 동일 Replication Domain 사용
```

## 관련 근거

- Issue #50: https://github.com/seokpan/seokpan-infra/issues/50
- PR #88: https://github.com/seokpan/seokpan-infra/pull/88
- 관련 Issue #51: https://github.com/seokpan/seokpan-infra/issues/51