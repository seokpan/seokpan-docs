[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-031 — MariaDB 백업 체인 상태(state)가 호스트 로컬에 종속되어 auto_failover 시 체인을 인식 못 하던 문제

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-10 (2026-09-09~10 야간 장애 재현) |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | MariaDB 백업 자동화(#55/#114/#129/#143), 정태훈(리뷰) |

## 최초 문제

`.backup_chain_state.json` 상태 파일과 `--incremental-basedir` 참조 경로가
호스트 로컬 디스크에 있었다. `auto_failover=true` 환경에서는 어느 호스트가
백업을 수행할지 스케줄과 무관하게 바뀔 수 있어, 실제로 수요일 Full을
mariadb-02가 수행한 뒤 failover가 발생하고 목요일 Incremental을 mariadb-01이
시도하면서 "이번 주 유효 체인 없음"으로 오판, Full로 승격을 시도하다
`mkdir` EPERM으로 백업 자체가 무산됐다.

## 조치

`.backup_chain_state.json`/`--incremental-basedir` 참조를 NFS 공유
스토리지로 이전해 두 호스트가 항상 같은 상태를 보도록 변경했다. 상태 파일
갱신의 임시파일도 `STATE_FILE`과 같은 디렉터리(NFS)에 만들도록 바꿔,
파일시스템 간 `mv`로 인한 원자성 손상을 막았다.

## 검증 (Test A/B)

- Test A: mariadb-02에서 Full 보유 중 강제 switchover → 이 체인을 만든 적
  없는 mariadb-01에서 Incremental 실행 → 무효 판정/승격 없이 정상 완주
- Test B(반대 방향): 다시 switchover 후 mariadb-02에서 Incremental 실행 →
  동일하게 정상 완주

## 리뷰에서 발견한 새 위험: 공유 상태의 동시 read-modify-write

리뷰어(정태훈)가 상태 authority를 NFS로 옮기면서 새로 생긴 위험을 지적했다.
두 호스트가 동시에 같은 이전 상태를 읽고 각자 갱신하면 나중에 쓰는 쪽이
먼저 쓴 갱신을 덮어쓸 수 있다(lost update).

`flock` 기반 공유 lock을 상태 읽기 직전부터 최종 상태 기록까지 전체
구간에 적용했다. lock 획득에 실패(타임아웃)하면 에러가 아니라 정상
스킵으로 처리한다 — 다른 실행이 이미 이번 사이클을 처리 중이라는 뜻이므로
중복 실행 없이 넘어가면 된다.

실제 mariadb-01/mariadb-02 양쪽에서 동일 시각에 프로덕션 lock 파일 경로로
`flock` 경쟁을 3회 반복해, 두 호스트가 동시에 lock을 획득한 사례가 없음을
확인했다.

## 부수 발견: 크로스 서브넷 NFSv4의 lock 폴링 지연

검증 중 mariadb-01(192.168.52.x)과 mariadb-02(192.168.51.x)가 서로 다른
서브넷에서 NFS 서버(192.168.54.x)를 마운트하고 있어, lock 해제가 콜백으로
즉시 전파되지 않고 약 30초 주기 폴링으로만 대기 측에 통지되는 특성을
발견했다. 안전성 문제는 아니었지만(동시 획득은 매 회차 막혔음), 기존 대기
시간(30초)이 이 폴링 주기와 거의 같아 짧게 점유된 lock조차 못 잡고 불필요한
스킵이 날 수 있었다. 대기 시간을 60초로 늘려 한 폴링 사이클 이상을
흡수하도록 조정했다.

## Before → After

```
Before
상태가 호스트 로컬 디스크에 종속
+ 여러 호스트의 동시 접근에 대한 보호 없음

After
상태를 NFS 공유 스토리지로 이전, 원자적 교체 보장
+ flock 기반 공유 lock으로 동시 read-modify-write 방지
+ 크로스 서브넷 폴링 지연을 반영한 대기 시간 조정
```

## 관련 근거

- Issue #166: https://github.com/seokpan/seokpan-infra/issues/166
- PR #168: https://github.com/seokpan/seokpan-infra/pull/168
- 상위 Issue #143, 관련 Issue #129/#114/#55