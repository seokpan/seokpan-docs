[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-020 — BuildKit Harbor Robot Credential 파일 계약 불일치

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-03 |
| **상태** | **해결** |
| **주 담당** | **최유준 — Delivery & Observability** |
| **영향 범위** | Jenkins Rootless BuildKit Agent Pod의 Harbor 인증 경로, CI/CD Build → Push 단계 |

## 선행 사건

이 문제는 [TS-017 — Jenkins Rootless BuildKit Harbor CA Trust 미적용 문제](TS-017_buildkit-harbor-ca-trust.md)를 해결한 뒤 Harbor 인증 단계까지 진행하면서 확인됐다.

TS-017과 동일한 `BuildKit → Harbor` Consumer 검증 흐름에서 연속으로 발견됐지만, CA Trust와 Credential 파일 계약은 Root Cause와 수정 위치가 서로 달라 별도 Troubleshooting으로 관리한다.

## 문제 개요

Harbor CA Trust 문제를 해결한 뒤 `ci-verify-buildkit-push`를 다시 실행하자 기존 x509 오류 대신 다음 단계에서 `401 Unauthorized`가 발생했다.

Harbor Robot Account Credential은 Kubernetes Secret `harbor-robot-dockerconfig`로 Agent Pod에 Mount되고 있었지만 BuildKit이 해당 Credential을 읽지 못해 인증 요청이 실패했다.

## 원인 분석

기존 Secret은 Kubernetes 표준 Docker Config Secret 형태였다.

```text
type: kubernetes.io/dockerconfigjson
data key: .dockerconfigjson
```

Jenkins BuildKit Container에는 다음 환경이 이미 설정돼 있었다.

```text
DOCKER_CONFIG=/home/user/.docker
```

그러나 BuildKit/Docker Auth Provider가 해당 디렉터리에서 기대하는 파일명은 다음이었다.

```text
/home/user/.docker/config.json
```

Secret Volume은 Secret의 Key 이름을 그대로 파일명으로 Mount하므로 기존 구조에서는 실제 파일이 다음처럼 생성됐다.

```text
/home/user/.docker/.dockerconfigjson
```

즉 Credential 값이나 Robot Account 자체가 잘못된 것이 아니라 **Kubernetes Secret의 파일명 계약과 BuildKit이 소비하는 Docker Config 파일명 계약이 일치하지 않은 것**이 원인이었다.

사전 검색에서 `harbor-robot-dockerconfig`는 BuildKit Agent용 Secret으로만 사용되고 있었고 kubelet `imagePullSecrets` 등 다른 표준 `dockerconfigjson` Consumer가 함께 사용하지 않는 것도 확인했다.

## 조치

`seokpan-infra`의 `jenkins_secrets` Ansible Role에서 Secret 구조를 BuildKit의 실제 소비 계약에 맞췄다.

```text
Before
type: kubernetes.io/dockerconfigjson
key: .dockerconfigjson

After
type: Opaque
key: config.json
```

Credential 내용과 Robot Account 값 자체는 변경하지 않았다.

Kubernetes Secret의 `type`은 기존 Resource에 대해 변경할 수 없는 필드이므로 최초 전환 시 기존 `harbor-robot-dockerconfig` Secret을 삭제한 뒤 새 구조로 재생성했다.

JCasC의 Secret Volume은 Secret 이름만 참조하고 내부 Key를 고정하지 않으므로 GitOps Volume 정의를 추가 수정할 필요는 없었다.

## 검증

- Repository 전체 검색으로 `harbor-robot-dockerconfig`가 BuildKit Agent 전용이고 `imagePullSecrets` 등 다른 용도로 겸용되지 않음을 확인
- 기존 Secret Type 변경 시 `field is immutable` 오류를 확인하고 Delete/Recreate 방식으로 전환
- 재생성 후 Secret `type: Opaque` 확인
- Secret Data Key가 `config.json`인지 확인
- BuildKit Agent Pod 내부 `/home/user/.docker/config.json` 실제 생성 확인
- Ansible Playbook 2차 재실행 `changed=0` 확인
- `ci-verify-buildkit-push` 재실행에서 기존 `401 Unauthorized` 미재발
- 실제 Harbor Push 성공
- Harbor에서 신규 Tag/Digest 등록 확인

## Before → After

```text
Before
harbor-robot-dockerconfig
→ kubernetes.io/dockerconfigjson
→ .dockerconfigjson으로 Mount
→ BuildKit은 config.json을 찾음
→ Credential 미인식
→ 401 Unauthorized

After
BuildKit 전용 Secret을 Opaque + config.json으로 구성
→ /home/user/.docker/config.json 생성
→ Robot Credential 인식
→ Harbor 인증/Push 성공
→ 재실행 changed=0
```

## 관련 사건

- 선행 CA Trust 문제: [TS-017 — Jenkins Rootless BuildKit Harbor CA Trust 미적용 문제](TS-017_buildkit-harbor-ca-trust.md)
- 두 사건은 같은 Pipeline 검증 흐름에서 연속으로 드러났지만 독립 Root Cause이므로 각각 별도 해결 사례로 유지한다.

## 관련 근거

- Infra Issue #126: https://github.com/seokpan/seokpan-infra/issues/126
- Infra PR #127: https://github.com/seokpan/seokpan-infra/pull/127
- GitOps Issue #21 — CoreDNS → CA Trust → Credential 순으로 이어진 Consumer 재검증 기록: https://github.com/seokpan/seokpan-gitops/issues/21
- GitOps PR #24 — 선행 CA Trust 정합화 및 Jenkins Root 연결: https://github.com/seokpan/seokpan-gitops/pull/24
