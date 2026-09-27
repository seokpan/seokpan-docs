[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-045 — 원 계획의 maxscale_exporter(공식 REST API exporter)가 존재하지 않아 소스 빌드 방식으로 2차 이관

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-18 |
| **상태** | **부분 해결 (REST 계정 코드화 완료, exporter는 2차 이관)** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | MaxScale 상태 Prometheus 수집 (운영 서버 영향 없음) |

## 최초 문제

이슈 #199/#211의 원 계획은 `mariadb-corporation/maxscale_exporter`(MaxScale REST API 기반)의 바이너리를
mysqld_exporter처럼 "공식 릴리스 SHA256 확인 후 배포"하는 것이었다.

## 원인

- 해당 공식 exporter는 MariaDB JIRA MXS-3022("Convert Prometheus-exporter to REST-api")가
  Resolution: Won't Do로 닫혀 있어 실제로 존재하지 않는다.
- 커뮤니티 대안(`Vetal1977/maxctrl_exporter` 및 포크)은 GitHub Release에 사전 빌드 바이너리가 없어
  `go build`가 필요하고, 신뢰할 수 있는 공식 체크섬 경로가 없다.

## 조치

- 신뢰할 수 없는 체크섬으로 임의 배포하지 않고 exporter 배포를 2차로 분리했다.
- 1차에서는 REST read-only 계정(listener `127.0.0.1:8989` 제한 + 전용 basic 계정)만 코드화했다(PR #214).
- 2차 절차(A안): 커밋 고정 → 소스 리뷰 → ansible-01 빌드 → SHA256 고정 → maxscale-01에는 바이너리만 배포.
- MaxScale 상태는 그때까지 `maxctrl list servers` 수동 확인으로 대체한다.

## 검증

- REST 계정: 1회차 changed=1 → 2회차 changed=0, 읽기(`/v1/servers`) 200, 쓰기 시도 401, 검증 전후 서버 상태 불변,
  Vault 값으로 REST 인증 200

## Before → After

Before
공식 exporter 바이너리 + SHA256 검증 배포(mysqld_exporter와 동일 패턴)

After
공식 exporter 부재 → 소스 빌드 + 자체 checksum 방식으로 2차 이관, 1차는 REST 계정만 확보

## 관련 근거

- Issue #211: https://github.com/seokpan/seokpan-infra/issues/211
- PR #214: https://github.com/seokpan/seokpan-infra/pull/214
- MXS-3022: https://jira.mariadb.org/browse/MXS-3022
