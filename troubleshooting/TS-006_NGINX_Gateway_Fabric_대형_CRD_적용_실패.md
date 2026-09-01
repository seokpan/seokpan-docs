[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-006 — NGINX Gateway Fabric 대형 CRD가 클라이언트 방식 적용(client-side apply) Annotation 제한에 걸림

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-28 |
| **상태** | **해결** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | 이유빈(HAProxy/Network), 최유준(Argo/GitOps), 애플리케이션 배포 경로 |

## 문제 개요

NGINX Gateway Fabric CRD를 일반 `kubectl apply`로 적용하는 과정에서:

```text
metadata.annotations: Too long
```

오류가 발생했다.

## 원인 분석

대형 CRD를 client-side apply하면 기존 객체 상태를 Annotation에 저장하는 과정에서 크기 제한에 걸릴 수 있었다.

즉 Manifest 자체의 기능 오류가 아니라 **적용 전략(client-side apply)의 한계**였다.

## 조치

- NGF 대형 CRD만 `kubectl apply --server-side` 사용
- 이미 생성된 일부 CRD는 삭제하지 않음
- CRD `Established` 대기
- Controller 배포
- `kubectl diff`로 실제 차이가 있을 때만 apply
- 서비스별 `NginxProxy`, `Gateway`, `HTTPRoute`는 GitOps 담당 범위로 분리

## 검증

- `nginx-gateway` Deployment `1/1`
- Controller Pod Running / restart 0
- `GatewayClass nginx`
  - `Accepted=True`
  - `SupportedVersion=True`
- 내부 TLS Secret 생성
- cert-generator Job Complete
- 동일 Playbook 재실행 `changed=0`, `failed=0`

## 설계적으로 얻은 결과

```text
Ansible
→ Gateway API/NGF CRD
→ NGF Controller
→ Platform GatewayClass

GitOps
→ NginxProxy
→ Gateway
→ HTTPRoute
→ 서비스별 Git에 선언한 목표 상태(Desired State)
```

초기 구성 자동화(Bootstrap)과 실제 실행 환경(Runtime)의 목표 상태(Desired State)를 어느 쪽이 관리하는지 실제 오류 해결 과정에서 명확히 했다.

## 관련 근거

- Infra Issue #12: https://github.com/seokpan/seokpan-infra/issues/12
- PR #48: https://github.com/seokpan/seokpan-infra/pull/48
- GitOps Gateway Issue #5: https://github.com/seokpan/seokpan-gitops/issues/5