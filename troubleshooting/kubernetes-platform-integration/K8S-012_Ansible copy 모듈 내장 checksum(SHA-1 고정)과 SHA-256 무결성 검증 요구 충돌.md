[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-039 — Ansible copy 모듈 내장 checksum(SHA-1 고정)과 SHA-256 무결성 검증 요구 충돌

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-15 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | DR-02 자동화(etcd_dr Role), 운영 서버 영향 없음 |

## 최초 문제
etcd Snapshot을 격리 3-node로 전달한 뒤 `ansible.builtin.copy` 모듈의
`checksum` 파라미터로 전송 무결성을 검증하려 했으나, DR-02 설계 기준인
SHA-256과 값이 맞지 않았다.

## 원인
`copy` 모듈의 `checksum` 파라미터는 내부적으로 SHA-1로 고정되어 있어,
SHA-256 기준으로 계산한 값과 비교하면 항상 불일치로 판정된다.

## 조치
`copy` 모듈의 내장 checksum 비교에 의존하지 않고, 전송 후 `stat` 모듈로
대상 파일의 SHA-256을 직접 계산한 뒤 `assert`로 원본 값과 비교하도록
별도 태스크를 구성했다.

## 검증
- 격리 3-node 각각에서 Snapshot SHA-256이 원본과 100% 일치함을
  `stat`+`assert` 태스크로 확인(PR #191 E2E 실행 결과)

## 관련 근거
- Issue #176: https://github.com/seokpan/seokpan-infra/issues/176
- PR #191: https://github.com/seokpan/seokpan-infra/pull/191
