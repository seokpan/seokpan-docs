[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-012 — Harbor VM 재부팅 후 대부분의 컨테이너가 자동 기동하지 않음

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-31 |
| **상태** | **해결** |
| **주 담당** | **최유준 — CI/CD 및 모니터링·관측** |
| **영향 범위** | 정태훈(애플리케이션·Kubernetes 이미지 사용), 전체 CI/CD |

## 문제 개요

Harbor VM을 재부팅했더니:

```text
harbor-log
```

만 실행되고:

```text
registry
registryctl
postgresql
core
portal
jobservice
redis
proxy
```

등은 모두 `Exited` 상태였다.

Harbor API도:

```text
HTTP 000
```

즉 응답하지 않았다.

## 수동 복구 확인

```bash
cd /opt/harbor/harbor
podman-compose -f docker-compose.yml up -d
```

후 전체 컨테이너가 `Up`으로 돌아왔고:

```text
https://harbor.seokpan.soldesk.store
→ HTTP 200
```

으로 복구됐다.

## 원인 분석

설치 스크립트는 Harbor를 올렸지만 **VM Boot 시 Compose Stack을 다시 올리는 Host-level Service**가 없었다.

즉:

```text
설치 성공
≠
재부팅 후 자동 복구
```

였다.

## 조치

`harbor.service` systemd Unit 추가.

핵심:

- After Network
- After Podman
- `podman-compose up -d`
- `RemainAfterExit=yes`
- `WantedBy=multi-user.target`

## 최종 검증

PR #69의 후속 승인 검토에 VM 재부팅 Evidence가 기록됐다.

재부팅 후:

- Harbor 전체 컨테이너 정상 기동
- HTTPS HTTP 200
- `harbor.service enabled`
- `harbor.service active`

확인 완료.

따라서 이 사례는 현재 **해결**로 기록할 수 있다.

## Before → After

```text
Before
VM Reboot
→ Harbor 대부분 Exited
→ API Down

After
systemd Unit
→ Boot 시 Compose Stack 복구
→ HTTPS 200
```

## 관련 근거

- Issue #62: https://github.com/seokpan/seokpan-infra/issues/62
- PR #69: https://github.com/seokpan/seokpan-infra/pull/69
- 재부팅 검증 승인 Review: https://github.com/seokpan/seokpan-infra/pull/69#pullrequestreview-5068702053
