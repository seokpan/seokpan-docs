[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-040 — Jinja2 JSON 파싱 시 `.items`가 dict.items() 메서드로 오인됨

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-15 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | DR-02 Object 검증 태스크(object_validate.yml), 운영 서버 영향 없음 |

## 최초 문제
Kubernetes Object 검증 단계에서 `kubectl get -o json` 결과를 파싱해
`namespace_json.items`처럼 접근했으나, 기대한 list 대신 다른 값이
반환되며 Object 개수 비교가 어긋났다.

## 원인
Jinja2 템플릿에서 `dict.items`는 JSON key `items`(Kubernetes List
Object의 표준 필드명)가 아니라 Python dict의 내장 메서드 `.items()`로
먼저 해석된다. 점(dot) 표기법이 속성 접근과 메서드 접근을 구분하지
못해 발생한 문제.

## 조치
`.items` 점 표기법 대신 `['items']` 대괄호 표기법으로 명시적으로
접근하도록 전체 Object 검증 태스크를 수정했다.

## 검증
- 수정 후 Namespace/Deployment/Secret 각각의 실제 개수(13/21/39)가
  baseline과 정확히 일치하는 것을 확인(PR #191 E2E 실행 결과)

## 관련 근거
- Issue #176: https://github.com/seokpan/seokpan-infra/issues/176
- PR #191: https://github.com/seokpan/seokpan-infra/pull/191
