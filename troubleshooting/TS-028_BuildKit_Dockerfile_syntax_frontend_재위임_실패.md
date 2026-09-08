[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-028 — Dockerfile syntax directive가 외부 frontend를 사용하게 해 Rootless BuildKit Build가 중단됨

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-04 |
| **상태** | **해결** |
| **주 담당** | **최유준 — CI/CD 및 모니터링·관측 / 정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합·검증** |
| **영향 범위** | Backend/Frontend Dockerfile, Jenkins Rootless BuildKit Image Build 경로 |

## 문제 개요

`seokpan-app` PR #48에서 Backend/Frontend Dockerfile을 Rootless BuildKit으로 검증하는 과정에서 Build가 Dockerfile 본문의 `RUN` 단계에 진입하기 전에 중단됐다.

당시 두 Dockerfile의 첫 줄에는 다음 syntax directive(Dockerfile에서 사용할 frontend를 지정하는 선언)가 있었다.

```dockerfile
# syntax=docker/dockerfile:1
```

해당 directive를 제거한 뒤에는 이전 실패 지점을 통과해 실제 Dockerfile `RUN` 단계까지 진행했으며, 이후 별개의 Rootless BuildKit 실행환경 문제인 TS-022가 다음 문제로 드러났다.

따라서 같은 Build 과정에서 연속으로 발생했지만 두 문제를 하나의 원인으로 처리하지 않았다.

## 원인 분석

Dockerfile의 syntax directive는 BuildKit이 사용할 Dockerfile frontend를 지정한다.

문제가 발생한 과거 Commit에서는 다음 흐름이 확인됐다.

```text
# syntax=docker/dockerfile:1
→ docker/dockerfile:1 frontend를 불러오는 단계로 진입
→ Dockerfile 1행에서 Build 중단
```

반면 해당 directive를 제거한 Commit에서는 같은 첫 번째 실패가 사라지고 Base Image와 Build Context를 처리한 뒤 실제 `RUN` 단계까지 진행했다.

따라서 이 사건은 Jenkins Agent의 seccomp 또는 process sandbox 설정 문제가 아니라 **Application Dockerfile이 외부 frontend를 사용하도록 지정한 데서 발생한 문제**였다.

## 조치

Backend와 Frontend Dockerfile에서 다음 directive를 제거했다.

```dockerfile
# syntax=docker/dockerfile:1
```

수정 Commit:

```text
9822183ce26f37684cf2003d7489c6199f9f880a
```

과거 Commit을 대조해 Backend/Frontend Dockerfile 모두 해당 directive 한 줄만 제거된 것을 확인했다.

## 검증

### 당시 작업 흐름

PR #48의 수정 이후 Rootless BuildKit 검증은 기존 frontend 실패 지점을 넘어 다음 단계로 진행했다.

이후 Dockerfile 첫 `RUN` 단계에서 `/proc mount ... operation not permitted` 오류가 발생했고, 이 문제는 Jenkins BuildKit Agent 설정을 수정한 TS-022로 별도 분리됐다.

최종적으로 TS-022 조치까지 적용된 Rootless BuildKit Agent에서 Backend/Frontend Build+Push가 모두 성공했다.

### 후속 A/B 재현

2026-09-08에는 현재 Jenkins/JCasC 또는 `seokpan-app main`을 되돌리지 않고, 과거 Commit과 당시 핵심 Rootless BuildKit 실행 조건을 이용해 격리된 환경에서 A/B 재현을 수행했다.

재구성한 핵심 조건:

```text
BuildKit: moby/buildkit:v0.32.2-rootless
buildctl: v0.32.2
runAsUser: 1000
BUILDKITD_FLAGS: unset
seccompProfile: Unconfined 미적용
```

이는 당시 Jenkins Agent 전체 상태를 1:1로 복원한 것이 아니라, 두 실패 지점의 순서와 원인 구분을 확인하기 위해 핵심 조건만 다시 구성한 후속 재현이다.

#### Before — syntax directive 존재

```text
#2 resolve image config for docker-image://docker.io/docker/dockerfile:1
...
Dockerfile:1
1 | >>> # syntax=docker/dockerfile:1
error: failed to solve: exit code: 1

BEFORE_RC=1
```

Dockerfile 본문의 실제 `RUN` 단계에 진입하기 전에 Build가 중단됐다.

#### After — syntax directive 제거

기존 frontend 실패가 사라지고 다음 단계까지 진행했다.

```text
Build Context load
Base Image resolve
첫 RUN 단계 진입
```

다음 실패는:

```text
error mounting "proc" to rootfs at "/proc":
operation not permitted

AFTER_RC=1
```

이었다.

이 오류는 기존 TS-022에서 기록한 Rootless BuildKit의 `/proc` 권한과 process sandbox 문제에 해당한다.

## Before → Change → After

```text
Before
# syntax=docker/dockerfile:1
→ 외부 Dockerfile frontend 사용
→ Dockerfile 1행에서 Build 중단

Change
Backend/Frontend Dockerfile에서 syntax directive 제거

After
기존 frontend 문제 해소
→ Build Context / Base Image 처리
→ 실제 첫 RUN 단계 진입
→ 별도 TS-022의 /proc 권한 문제 확인
```

## TS-022와의 사건 구분

두 사건은 같은 PR #48 Build 과정에서 연속으로 드러났지만 원인과 수정 위치가 다르다.

```text
TS-028
원인: Dockerfile의 외부 frontend 지정
수정: Backend/Frontend Dockerfile
실패 지점: Dockerfile frontend 처리 단계

TS-022
원인: Rootless BuildKit의 /proc 권한과 process sandbox 제약
수정: Jenkins BuildKit Agent 설정(JCasC)
실패 지점: Dockerfile 첫 RUN 단계
```

따라서 syntax directive 제거와 TS-022의 BuildKit Agent 설정 변경은 서로 다른 실패 지점을 해결한 독립 수정으로 기록한다.

## 관련 사건

- [TS-022 — Rootless BuildKit이 Dockerfile 첫 RUN에서 /proc 권한 문제로 실패](TS-022_BuildKit_Rootless_nested_RUN_seccomp_실행_실패.md)

TS-028의 Dockerfile frontend 문제가 제거된 뒤 TS-022의 첫 `RUN` 문제가 다음 단계에서 확인됐다.

## 관련 근거

- Docs Issue #68: https://github.com/seokpan/seokpan-docs/issues/68
- App PR #48: https://github.com/seokpan/seokpan-app/pull/48
- Before Commit: `bac7b785928fbbca7f462a694f7a28c5b823e707`
- Fix Commit: `9822183ce26f37684cf2003d7489c6199f9f880a`
- GitOps PR #34: https://github.com/seokpan/seokpan-gitops/pull/34
- Docs Issue #68 후속 A/B 재현 결과
