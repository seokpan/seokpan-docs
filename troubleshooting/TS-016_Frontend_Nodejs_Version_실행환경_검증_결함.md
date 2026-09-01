[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-016 — Frontend가 잘못된 Node.js 버전에서도 `npm ci`를 통과하던 실행환경 검증 결함

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-31 |
| **상태** | **해결** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | 최유준(Jenkins·컨테이너 빌드), 애플리케이션 개발·검증 참여자 |

## 문제 개요

애플리케이션 기본 구조(Application Scaffold) PR #7은 Frontend 공식 실행 기준을 다음처럼 정의했다.

```text
Node.js 24
npm 12.0.2
```

Backend는 `uv`의 `required-version`으로 잘못된 버전을 실행 단계에서 거부하도록 구성되어 있었다. Frontend도 `package.json`의 `engines`에 Node.js 24 기준을 선언했지만 별도 검증에서 공식 버전과 다른 환경이 사용되면서 실행 차단 규칙이 충분하지 않다는 점이 드러났다.

## 실제 발견

검증 환경:

```text
Node.js 22.22.2
npm 10.9.7
```

프로젝트 공식 기준:

```text
Node.js 24
npm 12.0.2
```

이 상태에서 `npm ci`를 실행하면 `EBADENGINE` 경고는 발생하지만 설치가 계속 진행됐고 TypeScript, Vitest, Vite Production Build까지 통과했다.

즉 다음 상태였다.

```text
package.json engines에 공식 Version 계약 존재
→ 잘못된 Node Version에서 경고 발생
→ 설치와 검증은 계속 진행
```

기능 장애는 아니었지만 프로젝트가 고정한 실행환경과 다른 버전에서도 검증 성공처럼 보일 수 있는 재현성 결함이었다.

## 원인 분석

`package.json`의 `engines` 선언만으로는 npm 기본 설정에서 항상 설치를 중단하지 않는다.

```text
Version 요구사항 선언
≠
Version 위반 시 강제 실패
```

따라서 Frontend에도 버전 불일치를 설치 단계에서 즉시 실패시키는 규칙이 필요했다.

## 조치

`frontend/.npmrc`에 다음 설정을 추가했다.

```ini
engine-strict=true
```

## 검증

공식 환경인 다음 조합에서 재검증했다.

```text
Node.js 24.19.0
npm 12.0.2
```

검증 항목:

- `engine-strict=true` 인식
- `npm ci`
- TypeScript Type Check
- Vitest
- Vite Production Build
- `npm audit`

모두 정상 통과했다.

새 Commit으로 기존 승인이 해제된 뒤 Pull Request의 최신 코드(Head)를 기준으로 다시 승인됐고 `main`에 Merge됐다.

## Before → After

| 구분 | Before | After |
|---|---|---|
| Node Version 계약 | `engines` 선언만 존재 | `.npmrc` `engine-strict=true` 추가 |
| 잘못된 Node 버전 | `EBADENGINE` 경고 후 계속 진행 가능 | 설치 단계에서 차단 가능 |
| 검증 신뢰도 | 기능 Test 통과만으로 Version mismatch를 놓칠 수 있음 | 실행환경 Version 계약도 검증 조건에 포함 |

## 해결 범위

이 사례에서 해결한 범위는 **Frontend 기본 구조의 Node/npm 실행환경 검증 규칙**이다. 이후 Jenkins·BuildKit·Container Image가 동일 버전을 사용하는지는 CI/CD 통합 단계의 별도 검증 대상이며, 이 보고서의 미완료 상태로 포함하지 않는다.

## 관련 근거

- Application Scaffold Issue #6: https://github.com/seokpan/seokpan-app/issues/6
- Application Scaffold PR #7: https://github.com/seokpan/seokpan-app/pull/7
- PR #7 별도 검증 Comment: https://github.com/seokpan/seokpan-app/pull/7#issuecomment-5479225246
- PR #7 후속 보완 Comment: https://github.com/seokpan/seokpan-app/pull/7#issuecomment-5479524358
