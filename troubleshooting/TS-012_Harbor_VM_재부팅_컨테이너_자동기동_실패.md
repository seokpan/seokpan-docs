[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-012 — Harbor VM 재부팅 후 대부분의 컨테이너가 자동 기동하지 않음

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 아래 내용만으로 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성했습니다.

| 항목 | 내용 |
|---|---|
| **시점** | 2026-08-31 |
| **상태** | **해결** |
| **주 담당** | **최유준 — CI/CD / 모니터링·관측** |
| **영향 역할** | 정태훈(Application/K8s Image), 전체 CI/CD |
| **핵심 범주** | Harbor / Docker Compose / systemd / Reboot Recovery |

## 장애 재현

Harbor VM 재부팅 후:

- `harbor-log` → Up / Healthy
- 나머지 8개 Container → `Exited (128)`

상태가 확인됐다.

개별 Container에는 `restart: always`가 설정돼 있었지만 전체 Harbor가 정상 복구되지 않았다.

## 수동 복구 확인

```bash
cd /root/harbor
docker compose up -d
```

실행 시 9개 Container가 모두 정상 기동했고:

```text
https://harbor.seokpan.soldesk.store → HTTP 200
```

을 확인했다.

## 원인 판단

Container 개별 Restart Policy만으로는 VM Boot 시 Compose `depends_on`과 전체 Startup Ordering을 신뢰성 있게 재현하지 못했다.

## 조치

`harbor.service` Systemd Unit을 도입했다.

주요 기준:

```text
After=docker.service network-online.target
Requires=docker.service
ExecStart=docker compose up -d
ExecStop=docker compose down
enabled
started
```

Container별 `restart: always`는 그대로 두고:

- 정상 운영 중 개별 Container 장애 → Docker Restart Policy
- VM Boot/Reboot 전체 Harbor Startup → systemd

로 역할을 분리했다.

## 최종 검증

수정 후 VM을 다시 재부팅해:

- Harbor 전체 Container 정상
- HTTPS 200
- `harbor.service` enabled / active

를 확인했다.

## Before → After

```text
Before
VM Reboot
  ↓
Docker 개별 restart policy
  ↓
Startup 순서 미보장
  ↓
8개 Container Exited(128)

After
VM Reboot
  ↓
systemd harbor.service
  ↓
docker/network 이후 compose up -d
  ↓
9개 Container 정상
  ↓
HTTPS 200
```

## 근거

- PR #74: https://github.com/seokpan/seokpan-infra/pull/74
- 최종 재부팅 검증/승인 Review: https://github.com/seokpan/seokpan-infra/pull/74#pullrequestreview-5064661305
- 관련 Issue #16: https://github.com/seokpan/seokpan-infra/issues/16

---
