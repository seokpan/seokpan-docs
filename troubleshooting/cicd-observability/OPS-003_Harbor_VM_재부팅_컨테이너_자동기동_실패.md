[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-010 — Harbor VM 재부팅 후 대부분의 컨테이너가 자동 기동하지 않음

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-31 |
| **상태** | **해결** |
| **주 담당** | **최유준 — CI/CD 및 모니터링·관측** |
| **영향 범위** | 정태훈(애플리케이션·Kubernetes 이미지 사용), 전체 CI/CD |

## 문제 개요

Harbor VM 재부팅 후 다음 상태가 확인됐다.

- `harbor-log` → Up / Healthy
- 나머지 8개 Container → `Exited (128)`

개별 Container에 `restart: always`가 설정되어 있었지만 Harbor 전체 서비스는 정상 복구되지 않았다.

## 수동 복구 확인

```bash
cd /root/harbor
docker compose up -d
```

실행 시 9개 Container가 모두 정상 기동했고 다음 주소가 HTTP 200을 반환했다.

```text
https://harbor.seokpan.soldesk.store
```

## 원인 분석

Container별 Restart Policy만으로는 VM Boot 시 Compose 의존관계와 전체 Startup 순서를 안정적으로 재현하지 못했다.

## 조치

`harbor.service` systemd Unit을 도입했다.

주요 기준:

```text
After=docker.service network-online.target
Requires=docker.service
ExecStart=docker compose up -d
ExecStop=docker compose down
enabled
started
```

역할을 다음처럼 분리했다.

- 정상 운영 중 개별 Container 장애 → Docker Restart Policy
- VM Boot/Reboot 시 Harbor 전체 시작 → systemd `harbor.service`

## 최종 검증

수정 후 Harbor VM을 다시 재부팅해 다음을 확인했다.

- Harbor 전체 Container 정상 기동
- HTTPS HTTP 200
- `harbor.service` enabled / active

## Before → After

```text
Before
VM Reboot
→ Docker 개별 restart policy에만 의존
→ Startup 순서 미보장
→ 8개 Container Exited(128)

After
VM Reboot
→ systemd harbor.service
→ docker/network 준비 후 compose up -d
→ 9개 Container 정상
→ HTTPS 200
```

## 관련 근거

- 관련 Issue #16: https://github.com/seokpan/seokpan-infra/issues/16
- PR #74: https://github.com/seokpan/seokpan-infra/pull/74
- 최종 재부팅 검증/승인 Review: https://github.com/seokpan/seokpan-infra/pull/74#pullrequestreview-5064661305
