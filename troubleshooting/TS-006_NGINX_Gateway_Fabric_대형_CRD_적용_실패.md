[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-006 — NGINX Gateway Fabric 대형 CRD가 client-side apply Annotation 제한에 걸림

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 아래 내용만으로 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성했습니다.

| 항목 | 내용 |
|---|---|
| **시점** | 2026-08-28 |
| **상태** | **해결** |
| **주 담당** | **정태훈 — Kubernetes / Kubernetes 플랫폼** |
| **영향 역할** | 이유빈(HAProxy/Network), 최유준(Argo/GitOps), Application |
| **핵심 범주** | Gateway API / CRD / Kubernetes Apply Strategy |

## 증상

NGINX Gateway Fabric CRD를 일반 `kubectl apply`로 적용하는 과정에서:

```text
metadata.annotations: Too long
```

오류가 발생했다.

## 원인

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
→ 서비스별 Desired State
```

Bootstrap과 Runtime Desired State의 소유권 경계를 실제 오류 해결 과정에서 명확히 했다.

## 근거

- Infra Issue #12: https://github.com/seokpan/seokpan-infra/issues/12
- PR #48: https://github.com/seokpan/seokpan-infra/pull/48
- GitOps Gateway Issue #5: https://github.com/seokpan/seokpan-gitops/issues/5

---
