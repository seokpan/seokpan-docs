[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-008 — Harbor Robot Account 자동화에서 잘못된 API 경로 404와 멱등성 분기 오류가 연쇄적으로 발견됨

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-28 ~ 2026-08-31 |
| **상태** | **해결** |
| **주 담당** | **최유준 — CI/CD 및 모니터링·관측** |
| **영향 범위** | Jenkins/BuildKit, Harbor 사용 경로 |

## 1단계 — Project별 Robot API 경로 가정이 실제 Harbor API와 달랐다

Harbor Robot Account 자동화의 최초 구현에서는 Harbor Project별 Robot Account API Endpoint를 다음과 같은 구조로 가정했다.

```text
/api/v2.0/projects/seokpan/robots
```

그러나 실제 Harbor 2.13.3에서 해당 경로는 `404`를 반환했다.

이 문제는 단순 통신 실패가 아니었다. **자동화 코드가 Harbor의 실제 Robot API 구조를 잘못 가정한 문제**였다.

## 2단계 — GC 정책 검증 중 Robot 존재 여부 분기 버그 추가 발견

후속 작업에서는 실제 Harbor API에 맞춰 Robot 조회 경로를 정리하고 자동화 검증을 계속했다.

그 과정에서 Project/Robot/GC 정책 검증 중 다음 문제가 다시 발견됐다.

Robot이 이미 존재하는 상태에서:

```yaml
set_fact:
  harbor_robot_exists: "{{ ... }}"
```

값이 문자열 형태로 취급되면서 실제 값이:

```text
"False"
```

인데도 Jinja truthiness에서는 non-empty string이므로 True처럼 동작할 수 있었다.

결과적으로:

- Robot이 없음
- 하지만 `not harbor_robot_exists`가 기대대로 동작하지 않음
- 생성 Task가 실행되지 않을 수 있음

이라는 멱등성 결함이 있었다.

## 최종 조치

다음 두 층의 문제를 각각 정리했다.

1. 실제 Harbor Robot API 구조에 맞게 조회 경로 수정
2. `set_fact`에 Boolean Jinja Expression 직접 사용

예:

```yaml
harbor_robot_exists: >-
  {{
    (
      harbor_robot_list_status.status | int == 200
      and harbor_robot_name in (...)
    )
  }}
```

## 검증

- Project 조회 성공
- Robot 조회 성공
- Robot 없는 상태에서 생성
- Robot 있는 상태에서 재생성 안 함
- 재실행 `changed=0`
- GC Schedule:

```text
type=WEEKLY
weekday=0
hour=3
```

- Garbage Collection Dry-run 성공

## Before → After

```text
Before 1
잘못된 Project Robot API
→ 404

After 1
실제 Harbor Robot API 기준으로 정리

Before 2
Robot 존재 여부를 문자열처럼 취급
→ 멱등성 분기 오류

After 2
실제 Boolean으로 계산
→ 없는 경우만 생성
→ 재실행 changed=0
```

## 관련 근거

- 초기 Robot API 404 기록: https://github.com/seokpan/seokpan-infra/issues/30
- PR #70: https://github.com/seokpan/seokpan-infra/pull/70
- PR #71: https://github.com/seokpan/seokpan-infra/pull/71