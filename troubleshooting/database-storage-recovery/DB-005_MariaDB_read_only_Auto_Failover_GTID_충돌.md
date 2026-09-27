[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-015 — MariaDB `read_only` 정적 설정이 자동 장애조치(Auto Failover)와 충돌하고 GTID Domain 설정도 잘못 분리됨

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-01 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | 정태훈(애플리케이션 DB 연동), 플랫폼 검증 |

## 최초 문제

`mariadb-02`의 `custom.cnf`에 다음 설정이 정적으로 들어 있었다.

```text
read_only=ON
```

자동 장애조치로 해당 서버가 Master가 되면 MaxScale이 실행 중에는 `read_only=OFF`로 바꿀 수 있지만, 서버 재부팅 시 설정 파일이 다시 적용되어 **Master인데 `read_only=ON`** 상태가 될 수 있었다.

## 1차 조치

정적 Config에서 `read_only`를 제거하고 MaxScale에 다음 기준을 적용했다.

```ini
enforce_read_only_slaves=true
```

즉 파일이 역할을 고정하지 않고, 실제 Master/Replica 역할에 따라 MaxScale이 `read_only` 상태를 관리하도록 변경했다.

## 실제 Failover 검증

### 첫 번째 강제 Master Reboot

Failover Threshold를 넘겨 실제 자동 장애조치가 발생했다.

```text
약 13초 후 mariadb-01 승격
새 Master read_only=OFF
```

### 두 번째 Reboot

Threshold 이내에 기존 Master가 복구되어 Failover 없이 역할을 유지했다.

```text
기존 Master 유지
read_only=OFF
```

Replica는 `read_only=ON`으로 정상 반영됐다.

## 추가 확인: Auto Rejoin이 GTID Divergence를 거부

첫 강제 Failover 과정에서 구 Master와 새 Master의 GTID가 갈라졌고 MaxScale은 이를 안전하지 않은 상태로 판단해 Auto Rejoin을 거부했다. 이는 장애가 아니라 데이터 정합성을 보호하기 위한 안전장치였다.

해당 서버는 `mariadb-backup`을 이용해 다시 구성하여 Cluster에 재편입했다. 이후 GTID가 일치하는 조건의 테스트에서는 Auto Rejoin이 정상 동작했다.

## 리뷰에서 발견한 GTID Domain 설정 오류

초기 PR에는 다음처럼 서버별로 다른 Domain이 지정돼 있었다.

```text
mariadb-01 → gtid_domain_id=1
mariadb-02 → gtid_domain_id=2
```

1 Primary + 1 Replica 구조에서 양쪽 서버가 자동으로 역할을 교환할 수 있으므로 두 서버는 같은 Replication Domain을 공유해야 했다.

최종 기준은 다음과 같이 수정했다.

```text
mariadb-01
server_id=1
gtid_domain_id=1

mariadb-02
server_id=2
gtid_domain_id=1
```

수정 설명만 믿지 않고 실제 Pull Request 최신 코드에서 양쪽 `gtid_domain_id: 1`이 반영됐는지도 다시 확인했다.

## 최종 검증

- Replica IO/SQL `Yes`
- 새 Write는 Domain 1
- 과거 Domain 2 이력은 더 증가하지 않음
- MaxScale에서 Master/Slave 역할 정상 표시
- Master `read_only=OFF`
- Slave `read_only=ON`
- Pull Request 최신 코드에서 양쪽 `gtid_domain_id: 1` 확인
- 최종 Review 승인
- PR #88 `main` Merge 완료

## Before → After

```text
Before
설정 파일이 read_only 역할을 고정
+ 서버별 GTID Domain 분리

After
실제 역할은 MaxScale이 관리
+ 동일 Replication Domain 사용
+ Failover/Rejoin 동작 재검증
```

## 관련 근거

- Issue #50: https://github.com/seokpan/seokpan-infra/issues/50
- PR #88: https://github.com/seokpan/seokpan-infra/pull/88
- 관련 Issue #51: https://github.com/seokpan/seokpan-infra/issues/51
