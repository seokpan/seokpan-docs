[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-007 — Harbor Robot Account 자동화에서 잘못된 API 경로 404와 멱등성 분기 오류가 연쇄적으로 발견됨

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-28 ~ 2026-08-31 |
| **상태** | **해결** |
| **주 담당** | **최유준 — CI/CD 및 모니터링·관측** |
| **영향 범위** | Jenkins/BuildKit, Harbor 사용 경로 |

## 1단계 — Robot API 경로 가정이 실제 Harbor API와 달랐다

초기 Robot Account 자동화는 Harbor Project별 API Endpoint를 전제로 구현하려 했으나 실제 Harbor v2 API에서 해당 경로가 동작하지 않아 **404**가 확인됐다.

```text
/api/v2.0/projects/{project}/robots
→ 404
```

확인 후 Robot 조회/생성 기준을 Harbor의 전역 Robot API인 다음 경로로 수정하고 이름과 Project를 기준으로 필터링하도록 바꿨다.

```text
/api/v2.0/robots
```

즉 문제는 Harbor 서비스 장애가 아니라 **API 구조를 잘못 가정한 자동화 구현 오류**였다.

## 2단계 — Robot 존재 여부 분기 오류 추가 발견

후속 Harbor GC 자동화를 검증하는 과정에서 `_harbor_robot_exists`가 Boolean이 아니라 문자열 `"False"` 형태로 전달될 경우 Jinja 조건에서 참으로 평가될 수 있는 문제가 발견됐다.

```text
"False" ≠ false
```

그 결과 존재하지 않는 Robot을 존재하는 것으로 판단해 생성 분기가 잘못될 수 있었다.

## 최종 조치

- 잘못된 Project별 Robot API Endpoint 제거
- 전역 `/api/v2.0/robots` 사용 후 이름/Project 기준 필터링
- `_harbor_robot_exists | bool` 명시적 변환
- 생성 과정에서 이미 존재하는 경우의 `409`를 허용 가능한 상태로 처리
- Robot Account 관련 민감한 값은 Git에 저장하지 않는 원칙 유지

## 검증

- Harbor API 호출 경로 정상 응답 확인
- Robot이 없는 상태에서 생성 정상
- Robot이 이미 있는 상태에서 중복 생성하지 않음
- 동일 Harbor Playbook 재실행 정상
- GC Schedule 생성/갱신 분기 재실행 검증
- Playbook `failed=0`

## Before → After

```text
Before
잘못된 Robot API 경로
→ 404
→ Endpoint 수정
→ 재실행에서 문자열 False 분기 오류 발견

After
실제 Harbor API 구조 사용
+ Boolean 명시 변환
+ 재실행 가능한 Robot/GC 자동화
```

## 관련 근거

- 초기 Harbor/Argo 자동화 PR #47: https://github.com/seokpan/seokpan-infra/pull/47
- Robot/GC 후속 PR #69: https://github.com/seokpan/seokpan-infra/pull/69
