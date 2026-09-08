[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-020 — BuildKit이 Harbor Robot 인증 파일을 찾지 못해 401 Unauthorized 발생

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-03 |
| **상태** | **해결** |
| **주 담당** | **최유준 — CI/CD 및 모니터링·관측** |
| **영향 범위** | Jenkins Rootless BuildKit Agent Pod의 Harbor 인증 경로, CI/CD Build → Push 단계 |

## 선행 사건

이 문제는 [TS-017 — Jenkins Rootless BuildKit이 Harbor 내부 CA를 신뢰하지 못해 Push가 실패](TS-017_buildkit-harbor-ca-trust.md)를 해결한 뒤 Harbor 인증 단계까지 진행하면서 확인됐다.

TS-017과 같은 BuildKit → Harbor 검증 과정에서 연속으로 발견됐지만, TS-017은 TLS 인증서 신뢰 문제이고 이번 사건은 BuildKit이 인증 파일을 찾지 못한 문제라 원인과 수정 위치가 서로 다르다.

## 문제 개요

Harbor 내부 CA 신뢰 문제를 해결한 뒤 `ci-verify-buildkit-push`를 다시 실행하자 기존 x509 오류 대신 다음 단계에서 `401 Unauthorized`가 발생했다.

Harbor Robot Account 인증정보는 Kubernetes Secret `harbor-robot-dockerconfig`로 Agent Pod에 마운트되어 있었지만 BuildKit이 해당 파일을 읽지 못해 인증 요청이 실패했다.

## 원인 분석

기존 Secret은 Kubernetes 표준 Docker Config Secret 형식이었다.

```text
type: kubernetes.io/dockerconfigjson
data key: .dockerconfigjson
```

Jenkins BuildKit Container에는 다음 환경변수가 설정되어 있었다.

```text
DOCKER_CONFIG=/home/user/.docker
```

BuildKit/Docker 인증 기능이 해당 디렉터리에서 찾는 파일명은 다음이었다.

```text
/home/user/.docker/config.json
```

하지만 Secret Volume은 Secret의 Key 이름을 그대로 파일명으로 만들기 때문에 기존 구조에서는 실제 파일이 다음처럼 생성됐다.

```text
/home/user/.docker/.dockerconfigjson
```

즉 Robot Account의 ID나 Password 값이 잘못된 것이 아니라 **Pod에 생성된 인증 파일 이름과 BuildKit이 찾는 파일 이름이 달랐던 것**이 원인이었다.

Repository 전체 검색 결과 `harbor-robot-dockerconfig`는 BuildKit Agent 전용 Secret으로 사용되고 있었고, kubelet의 `imagePullSecrets`처럼 표준 `dockerconfigjson` 형식을 요구하는 다른 용도로 함께 사용되지 않는 것도 확인했다.

## 조치

`seokpan-infra`의 `jenkins_secrets` Ansible Role에서 Secret 구조를 BuildKit이 실제로 읽는 파일 형식에 맞췄다.

```text
Before
type: kubernetes.io/dockerconfigjson
key: .dockerconfigjson

After
type: Opaque
key: config.json
```

Robot Account의 인증정보 자체는 변경하지 않았다.

Kubernetes Secret의 `type`은 기존 Resource에서 변경할 수 없는 필드이므로 최초 전환 시 기존 `harbor-robot-dockerconfig` Secret을 삭제한 뒤 새 구조로 다시 생성했다.

JCasC의 Secret Volume은 Secret 이름만 참조하고 내부 Key 이름을 고정하지 않으므로 GitOps Volume 정의는 추가로 수정할 필요가 없었다.

## 검증

- Repository 전체 검색으로 `harbor-robot-dockerconfig`가 BuildKit Agent 전용이고 `imagePullSecrets` 등 다른 용도로 함께 사용되지 않음을 확인
- 기존 Secret Type 변경 시 `field is immutable` 오류를 확인하고 Delete/Recreate 방식으로 전환
- 재생성 후 Secret `type: Opaque` 확인
- Secret Data Key가 `config.json`인지 확인
- BuildKit Agent Pod 내부에 `/home/user/.docker/config.json`이 실제 생성됐는지 확인
- Ansible Playbook 2차 재실행 `changed=0` 확인
- `ci-verify-buildkit-push` 재실행에서 기존 `401 Unauthorized` 미재발
- 실제 Harbor Push 성공
- Harbor에서 신규 Tag/Digest 등록 확인

## Before → After

```text
Before
harbor-robot-dockerconfig
→ kubernetes.io/dockerconfigjson
→ .dockerconfigjson 파일로 마운트
→ BuildKit은 config.json을 찾음
→ 인증정보를 읽지 못함
→ 401 Unauthorized

After
BuildKit 전용 Secret을 Opaque + config.json으로 구성
→ /home/user/.docker/config.json 생성
→ Robot Account 인증정보 인식
→ Harbor 인증/Push 성공
→ 재실행 changed=0
```

## 관련 사건

- 선행 CA 문제: [TS-017 — Jenkins Rootless BuildKit이 Harbor 내부 CA를 신뢰하지 못해 Push가 실패](TS-017_buildkit-harbor-ca-trust.md)
- 두 사건은 같은 BuildKit → Harbor 검증 과정에서 연속으로 드러났지만 원인과 수정 위치가 다르므로 각각 별도 사례로 유지한다.

## 관련 근거

- Infra Issue #126: https://github.com/seokpan/seokpan-infra/issues/126
- Infra PR #127: https://github.com/seokpan/seokpan-infra/pull/127
- GitOps Issue #21 — CoreDNS → CA 신뢰 → Robot Account 인증 순으로 이어진 재검증 기록: https://github.com/seokpan/seokpan-gitops/issues/21
- GitOps PR #24 — 선행 CA 문제 수정 및 Jenkins Root 연결: https://github.com/seokpan/seokpan-gitops/pull/24
