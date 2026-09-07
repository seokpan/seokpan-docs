[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-027 — Harbor Tag Immutability 미적용으로 동일 Tag의 Image Digest 덮어쓰기가 허용됨

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-02 ~ 2026-09-03 |
| **상태** | **해결** |
| **주 담당** | **정태훈 — Runtime Platform·통합** |
| **영향 범위** | Harbor `seokpan` Project, Jenkins/BuildKit Image Push 경로, Git Commit 기반 Image 추적성 |

## 문제 개요

Harbor `seokpan` Project는 존재하고 Private Project 정책도 적용되어 있었지만, 06 설계에서 요구한 **Tag Immutability**는 실제 자동화에 구현되지 않은 상태였다.

실제 Consumer 검증에서 이미 사용한 Tag에 다른 Image Manifest Digest를 다시 Push했을 때 Harbor가 이를 거부하지 않고 수락할 수 있음을 확인했다.

```text
기존 Tag
seokpan/test-push:credential-file-verify

문제 상태
동일 Tag + 다른 Manifest Digest 재Push
→ Harbor가 overwrite 허용
```

이 상태에서는 Git Commit과 Image Tag를 연결해도 Registry에서 기존 Tag가 다른 Digest로 바뀔 수 있어, 동일 Tag가 항상 같은 Artifact를 가리킨다는 전제를 보장할 수 없었다.

## Root Cause

Harbor Project 자동화에는 Project 생성·Private 설정 등은 구현되어 있었지만 Tag Immutability는 TODO 상태로 남아 있었다.

즉 정책을 문서에서 요구했지만 실제 Harbor Project Desired State에 다음 규칙이 없었다.

```text
Repository: **
Tag:        **
Action:     immutable
```

문제의 핵심은 Harbor 자체 오류가 아니라 **Registry 정책을 실제 자동화 Desired State에 반영하지 않은 구현 Gap**이었다.

## 조치

Infra #109 / PR #110에서 Harbor `seokpan` Project Tag Immutability를 Ansible로 관리하도록 구현했다.

### 정책

```text
Repository: **
Tag:        **
Action:     immutable
```

### 자동화 동작

- 기존 동일 Rule 조회
- 동일 Rule 존재 시 중복 생성하지 않음
- Disabled Rule이면 활성화
- 적용 후 활성 Rule 재조회 및 검증
- Harbor HTTPS API를 Controller-local에서 호출해 Project Policy를 SSH와 독립적으로 관리
- Harbor API Credential을 사용하는 Task는 `no_log: true` 적용

Harbor 설치·서비스 관리는 기존 SSH 기반 Role을 유지하고, Project Immutability 정책은 HTTPS API 기반 별도 자동화 경로로 분리했다.

## CI/CD Tag 정책과의 정합성

전체 Repository/Tag 범위에 Immutability를 적용하면 `latest` 같은 부동 Tag를 계속 덮어쓰는 Pipeline과는 충돌할 수 있다.

PR #110 리뷰에서 이 점을 확인했고, 현재 CI/CD 방향은 다음과 같이 정리되어 있었다.

```text
Image Tag
→ git-<commit-sha>
→ Commit마다 새 Tag 생성
→ 기존 Tag overwrite를 운영 방식으로 사용하지 않음
```

따라서 현재 Project의 Commit 기반 Tag 정책과 `** / **` Immutability는 충돌하지 않는 것으로 판단했다.

## 검증

### 1. 동일 Tag overwrite 거부

정책 적용 후 기존 Tag에 다른 Manifest Digest를 재Push했다.

대상 Tag:

```text
seokpan/test-push:credential-file-verify
```

재Push한 Manifest Digest:

```text
sha256:2e60caf5984df8fd6eb88de7075017d396c9a017f64cfaf575b7af7d43132f2d
```

Harbor 응답:

```text
Failed to process request due to
'test-push:credential-file-verify' configured as immutable.
```

결과:

```text
PASS — 동일 Tag overwrite 거부
```

### 2. 신규 Tag Push

신규 Tag:

```text
immutability-new-tag-verify
```

Manifest Digest:

```text
sha256:41f2e1852392f3bcf2b942353818884fc3e34d774f66b709137d2569e9b2b028
```

결과:

```text
PASS — 신규 Tag Push 정상 완료
```

즉 기존 Tag 변경만 차단하고 새로운 Tag 생성은 정상 허용하는 정책으로 동작했다.

### 3. Ansible 멱등성

동일 설정으로 재실행했다.

```text
changed=0
failed=0
unreachable=0
```

동일 Immutability Rule이 중복 생성되지 않고 Desired State에 수렴함을 확인했다.

## Before → Change → After

```text
Before
Harbor seokpan Project 존재
→ Tag Immutability 미구현
→ 동일 Tag가 다른 Digest를 가리킬 수 있음
→ Artifact 추적성 약화

Change
Harbor Project Tag Immutability 자동화
→ Repository ** / Tag ** / immutable
→ 기존 Rule 조회·중복 방지·활성화 검증

After
동일 Tag + 다른 Digest 재Push 거부
→ 신규 Tag Push 정상
→ Ansible 재실행 changed=0
→ Git Commit 기반 immutable Tag 운영과 정합
```

## 기존 Harbor 관련 TS와의 사건 경계

다음 사건들과 Harbor라는 공통 영역은 있지만 Root Cause가 다르다.

- TS-007 — Harbor Robot API 404·멱등성 분기 오류
- TS-017 — Jenkins Rootless BuildKit Harbor CA Trust
- TS-020 — BuildKit Harbor Credential 파일 계약 불일치
- TS-024 — Kubernetes containerd Harbor CA Trust

TS-027은 **TLS Trust·Credential·Robot API가 아니라 Registry Project의 Artifact 불변성 정책 미구현**을 다룬다.

## 관련 근거

- Docs Issue #66: https://github.com/seokpan/seokpan-docs/issues/66
- Infra Issue #109: https://github.com/seokpan/seokpan-infra/issues/109
- Infra PR #110: https://github.com/seokpan/seokpan-infra/pull/110
- Infra PR #110 Merge Commit: `03d38b2ff1898ac6cbbee6b50a466969c4138904`
- CI/CD Tag 정책 연계: https://github.com/seokpan/seokpan-gitops/issues/21
