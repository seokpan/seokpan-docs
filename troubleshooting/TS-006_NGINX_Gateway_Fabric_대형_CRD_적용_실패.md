[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-006 — NGINX Gateway Fabric 대형 CRD가 클라이언트 방식 적용(client-side apply) Annotation 제한에 걸림

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-28 |
| **상태** | **해결** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | 이유빈(HAProxy/Network), 최유준(Argo/GitOps), Application |

## 문제 개요

NGINX Gateway Fabric CRD 설치 중:

```text
CustomResourceDefinition ... is invalid:
metadata.annotations: Too long: may not be more than 262144 bytes
```

## 원인 분석

단순 설치 실패가 아니라 **대형 CRD + client-side apply의 last-applied annotation 크기 제한**이었다.

## 조치

정적 CRD 설치 방식을 `server_side`로 변경했다.

중요한 부분은 모든 CRD를 무조건 Server-Side Apply로 바꾼 것이 아니다.

- Gateway API Standard CRD → 일반 적용
- NGINX Gateway Fabric 대형 CRD → Server-Side Apply

즉 실패 원인이 확인된 범위에만 최소 수정했다.

## 검증

Version Lock 환경의 `kubernetes.core`에서도 다시 검증했다.

```text
Gateway API Standard: 5개 CRD
NGF: 8개 CRD
```

확인 완료.

## 설계적으로 얻은 결과

이후 역할 경계가 정리됐다.

**Ansible**

- CRD
- Controller
- GatewayClass

**GitOps**

- NginxProxy
- Gateway
- HTTPRoute
- 실제 서비스 Git에 선언한 목표 상태(Desired State)

즉 초기 구성 자동화(Bootstrap)과 실제 클러스터 실행 환경(Runtime) Desired State의 소유권도 함께 정리됐다.

## 관련 근거

- Issue #42: https://github.com/seokpan/seokpan-infra/issues/42
- PR #43: https://github.com/seokpan/seokpan-infra/pull/43
