[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-020 — Frontend가 잘못된 Node.js 버전에서도 `npm ci`를 통과하던 실행환경 검증 결함

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-31 |
| **상태** | **해결** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | 최유준(Jenkins·컨테이너 빌드), 애플리케이션 개발·검증 참여자 |

## 문제 개요

애플리케이션 기본 구조(Application Scaffold) PR #7은 Frontend의 공식 실행 기준을 다음처럼 고정했다.

```text
Node.js 24
npm 12.0.2
```

Backend 쪽은 `uv`의 `required-version`을 이용해 잘못된 `uv` 버전을 실제 실행 단계에서 거부하도록 구성되어 있었다. Frontend도 `package.json`의 `engines`에 Node.js 24 계열 기준을 선언했지만, 별도 팀원 검증에서 프로젝트 공식 버전과 다른 환경이 사용되면서 실행 Gate가 충분히 강하지 않다는 점이 드러났다.

## 실제 발견

리뷰어 환경:

```text
Node.js 22.22.2
npm 10.9.7
```

프로젝트 공식 기준:

```text
Node.js 24
npm 12.0.2
```

이 상태에서 `npm ci`를 실행했을 때 `EBADENGINE` 경고는 발생했지만 설치 자체는 계속 진행됐고, TypeScript/Vitest/Vite Build까지 통과했다.

즉:

```text
package.json engines에 공식 Version 계약 존재
→ 잘못된 Node Version에서 경고 발생
→ 하지만 설치/검증 계속 가능
```

이었다.

이는 기능 장애가 발생한 사건은 아니지만, **프로젝트가 고정한 실행환경과 다른 버전으로도 검증이 성공한 것처럼 보일 수 있는 재현성 결함**이다. 향후 로컬 개발자·Jenkins Build Agent·Container Build 환경이 달라질 경우 같은 Lockfile을 사용하더라도 실행환경 계약이 느슨해질 수 있다.

## 원인 분석

`package.json`의 `engines` 선언만으로는 npm이 기본 설정에서 항상 설치를 중단하지 않는다.

따라서 당시 구조는:

```text
Version 요구사항 선언
≠
Version 위반 시 강제 실패
```

였다.

Backend의 `uv required-version`처럼 Frontend에도 실제 Fail-fast Gate가 필요했다.

## 조치

리뷰 결과를 반영해 `frontend/.npmrc`에 다음 설정을 추가했다.

```ini
engine-strict=true
```

이후 공식 환경인:

```text
Node.js 24.19.0
npm 12.0.2
```

에서 다시 다음 항목을 검증했다.

- `engine-strict=true` 인식
- `npm ci`
- TypeScript Type Check
- Vitest
- Vite Production Build
- `npm audit`

모두 정상 통과했다.

## 검토 흐름

```text
초기 Scaffold 구현
→ 별도 팀원 재현 검증
→ Node 22/npm 10에서도 npm ci가 경고만 내고 진행됨
→ Version 계약 강제력이 부족하다는 사실 확인
→ engine-strict=true 추가
→ 공식 Node 24/npm 12 환경에서 재검증
→ 최신 Head 재승인
→ main Merge
```

새 Commit으로 기존 승인이 stale/dismissed된 뒤 최신 Head 기준으로 다시 승인된 점도 확인된다. 따라서 이 사례는 **리뷰가 단순 확인 절차가 아니라 실행환경 재현성 결함을 찾아 실제 Gate를 강화한 사례**다.

## Before → Change → After

| 구분 | Before | Change | After |
|---|---|---|---|
| Node Version 계약 | `engines` 선언 | `.npmrc` `engine-strict=true` | 잘못된 Engine을 설치 단계에서 차단 가능 |
| 비공식 Node 22 검증 | `EBADENGINE` 경고 후 계속 진행 | Fail-fast 정책 보강 | 공식 Node 24 계약을 실행 수준에서 강제 |
| 검증 신뢰도 | 기능 Test가 통과하면 Version mismatch를 놓칠 수 있음 | Version Gate를 별도 실패 조건으로 추가 | Lock + 실제 실행 환경(Runtime) Version 계약을 함께 검증 |

## 담당 역할 및 영향

- **정태훈**: Application 실행 기준·Scaffold Owner. Frontend의 재현 가능한 개발/검증 환경을 보장해야 함.
- **최유준**: 이후 Jenkins/BuildKit Pipeline이 Frontend Build를 소비하므로, CI 환경에서도 동일 Node Version 계약을 지켜야 함.
- **Application 개발 참여자**: 로컬 환경 차이로 “내 PC에서는 빌드됨”이 공식 P0/P1 Evidence로 오인되는 것을 방지.

## 남은 범위

이 사례로 확인한 것은 **로컬 Scaffold의 Node/npm 실행 Gate**다.

다음은 이후 CI/CD·Container Image 빌드 단계에서 실제 확인해야 한다.

- Frontend Container Image의 Node Build Stage Version
- Jenkins Agent/BuildKit에서 동일 Node Version 사용 여부
- CI에서 `npm ci`가 Version mismatch 시 실제 Fail하는지

따라서 Scaffold 단계의 문제는 해결됐지만, 이 결과를 Jenkins/Container 통합 완료 증거로 확장해서 쓰지는 않는다.

## 관련 근거

- Application Scaffold Issue #6: https://github.com/seokpan/seokpan-app/issues/6
- Application Scaffold PR #7: https://github.com/seokpan/seokpan-app/pull/7
- PR #7 별도 검증 Comment: https://github.com/seokpan/seokpan-app/pull/7#issuecomment-5479225246
- PR #7 후속 보완 Comment: https://github.com/seokpan/seokpan-app/pull/7#issuecomment-5479524358
