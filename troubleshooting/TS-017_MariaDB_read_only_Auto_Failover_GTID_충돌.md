[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-017 — MariaDB `read_only` 정적 설정이 Auto Failover와 충돌하고 GTID Domain 설정도 잘못 분리됨

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 아래 내용만으로 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성했습니다.

| 항목 | 내용 |
|---|---|
| **시점** | 2026-09-01 |
| **상태** | **핵심 해결 · Manual Switchover 후속 있음** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 역할** | 정태훈(Application DB), 리뷰를 통한 Platform Validation |
| **핵심 범주** | MariaDB / MaxScale / Auto Failover / GTID / read_only |

## 최초 문제

`mariadb-02`의 `custom.cnf`에:

```text
read_only=ON
```

이 정적으로 고정돼 있었다.

그러나 MaxScale `auto_failover=true` 환경에서 `mariadb-02`가 Master로 승격될 수 있다.

그 상태에서 재시작하면 설정 파일의 `read_only=ON`이 다시 적용되어:

```text
Master인데 쓰기가 막히는 상태
```

가 될 수 있었다.

## 1차 조치

- MariaDB `custom.cnf`에서 `read_only` 자체를 제거
- MaxScale Monitor:
  ```text
  enforce_read_only_slaves=true
  ```
  적용
- Runtime Role에 따라 MaxScale이 `read_only`를 관리
- `serial: 1`로 MariaDB 동시 재시작 방지
- MaxScale/Replication Password는 Vault로 이동

## 실제 Failover 검증

## 첫 번째 강제 Master Reboot

다운타임이 Failover 임계치인 약 8초를 넘음.

결과:

- Auto Failover 실제 발동
- `mariadb-01`로 승격
- 약 **13초**
- 새 Master `read_only=OFF`

## 두 번째 Reboot

다운타임이 임계치보다 짧아 Failover가 발생하지 않음.

결과:

- 기존 Master 자력 복구
- 재시작 후 `read_only=OFF`

## Replica

재시작 후:

```text
read_only=ON
```

으로 정상 수렴.

## 추가 발견: Auto Rejoin이 GTID Divergence에서 거부됨

첫 Failover 시험에서는 Old Primary의 Auto Rejoin이 GTID Divergence 때문에 실패했다.

MaxScale이 무리하게 붙이지 않고 안전하게 거부했고, `mariadb-backup` 기반 수동 재구축으로 다시 편입했다.

두 번째는 GTID가 일치한 상태에서 Auto Rejoin 성공.

## 리뷰에서 추가 발견: 서버별 `gtid_domain_id=1/2`는 현재 Topology와 맞지 않음

초기 PR 코드는:

```text
mariadb-01
server-id=1
gtid_domain_id=1

mariadb-02
server-id=2
gtid_domain_id=2
```

였다.

`server-id`가 서버별 고유해야 하는 것은 맞지만, 현재 구조는 동시 Multi-Primary가 아니라 **1-Primary / 1-Replica + Auto Failover**다.

서버별 Domain을 다르게 고정하면 승격에 따라 새 Write의 GTID Domain 자체가 바뀔 수 있다.

이 문제는 리뷰에서 차단했고 수정 요청했다.

## 최종 수정

```text
mariadb-01
server-id=1
gtid_domain_id=1

mariadb-02
server-id=2
gtid_domain_id=1
```

로 통일.

## 재검증

- 양쪽 `gtid_domain_id=1`
- `Slave_IO_Running: Yes`
- `Slave_SQL_Running: Yes`
- IO/SQL Error 없음
- 과거 Domain 2 GTID는 History로 남지만 더 증가하지 않음
- 새 Write는 Domain 1로만 증가
- `maxctrl list servers` 기준 양쪽 GTID 일치
- Master `read_only=OFF`
- Slave `read_only=ON`

이 검증 후 최신 PR 코드에 양쪽 `gtid_domain_id=1`이 실제 반영된 것을 다시 확인했고, 2026-09-01 최종 리뷰에서 승인되었다. 이후 PR #88은 `main`에 Merge되었다.

중간에는 “수정했다”는 코멘트가 먼저 올라왔지만 실제 PR Head에는 `mariadb-02`의 `gtid_domain_id: 2`가 남아 있어 즉시 승인하지 않았다. **설명/코멘트가 아니라 실제 Diff를 다시 확인한 뒤** 수정 Commit 반영 후 승인했다는 점도 이 사례의 중요한 검증 기록이다.

## 남은 문제: Manual Switchover Timeout

다음 명령:

```text
maxctrl call command mariadbmon switchover MariaDB-Monitor --timeout 30s
```

을 두 번 실행했으나 Client Timeout이 발생했고 실제 Role 전환도 없었다.

자동 Failover는 정상 검증됐지만 수동 Switchover 신뢰성은 별도 후속 문제다.

따라서 TS-017은:

- **Auto Failover/read_only/GTID 문제 = 해결**
- **Manual Switchover = 별도 후속 필요**

로 기록한다.

## Before → After

```text
Before
Host별 read_only 정적 고정
+ 서버별 GTID Domain 1/2
        ↓
Runtime Role이 바뀌어도 파일은 과거 역할 유지
        ↓
승격 후 Write 차단/GTID Stream 복잡화 위험

After
read_only는 MaxScale Runtime Role 기준 관리
+ server-id만 고유
+ GTID Domain은 동일
        ↓
강제 Reboot/Failover/Replica Restart 검증
        ↓
역할과 read_only 상태 정상 수렴
```

## 근거

- Issue #50: https://github.com/seokpan/seokpan-infra/issues/50
- PR #88: https://github.com/seokpan/seokpan-infra/pull/88

---
