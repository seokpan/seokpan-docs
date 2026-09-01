[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-005 — NGINX Gateway Fabric 대형 CRD가 클라이언트 방식 적용(client-side apply) Annotation 제한에 걸림

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-28 |
| **상태** | **해결** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | 이유빈(HAProxy/Network), 최유준(Argo/GitOps), 애플리케이션 배포 경로 |

## 문제 개요

NGINX Gateway Fabric CRD를 일반 `kubectl apply`로 적용하는 과정에서 다음 오류가 발생했다.

```text
metadata.annotations: Too long
```

## 원인 분석

대형 CRD를 client-side apply하면 기존 객체 상태를 Annotation에 저장하는 과정에서 크기 제한에 걸릴 수 있었다. Manifest 자체의 기능 오류가 아니라 **적용 방식(client-side apply)의 한계**였다.

## 조치

- NGINX Gateway Fabric의 대형 CRD는 `kubectl apply --server-side` 사용
- 이미 생성된 일부 CRD는 불필요하게 삭제하지 않음
- CRD `Established` 상태 대기
- Controller 배포 후 정상 상태 확인
- `kubectl diff`로 실제 차이가 있을 때만 apply
- 서비스별 `NginxProxy`, `Gateway`, `HTTPRoute`는 GitOps 관리 영역으로 분리

## 검증

- `nginx-gateway` Deployment `1/1`
- Controller Pod Running / restart 0
- `GatewayClass nginx`
  - `Accepted=True`
  - `SupportedVersion=True`
- 인증서 관련 리소스 생성 확인
- cert-generator Job Complete
- 동일 Playbook 재실행 `changed=0`, `failed=0`

## 설계에 반영한 내용

```text
Ansible
→ Gateway API / NGINX Gateway Fabric CRD
→ NGINX Gateway Fabric Controller
→ 공용 GatewayClass

GitOps
→ NginxProxy
→ Gateway
→ HTTPRoute
→ 서비스별 배포 상태
```

이 장애를 해결하면서 초기 구성 자동화(Bootstrap)와 서비스 실행 상태를 GitOps가 각각 어디까지 관리할지 경계를 명확히 했다.

## 관련 근거

- Infra Issue #12: https://github.com/seokpan/seokpan-infra/issues/12
- PR #48: https://github.com/seokpan/seokpan-infra/pull/48
- GitOps Gateway Issue #5: https://github.com/seokpan/seokpan-gitops/issues/5
