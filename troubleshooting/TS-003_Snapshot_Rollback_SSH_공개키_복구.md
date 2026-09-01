[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-003 — Snapshot Rollback 이후 SSH 공개키가 과거 상태로 돌아갈 수 있는 문제

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 아래 내용만으로 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성했습니다.

| 항목 | 내용 |
|---|---|
| **시점** | 2026-08-27 최초 기록 |
| **상태** | **진행 중** |
| **주 담당** | **이유빈 — 네트워크 및 공통 인프라·Ansible 통합** |
| **영향 역할** | 모든 팀원 |
| **핵심 범주** | Ansible Access / Recovery / Snapshot |

## 문제

VM을 SSH 공개키 배포 이전 Snapshot으로 되돌리면 `/root/.ssh/authorized_keys`도 과거 상태로 돌아갈 수 있다.

결과적으로:

- 어떤 팀원은 SSH 가능
- 어떤 팀원은 SSH 불가
- VM별로 상태가 다름
- Ansible Controller 사용자별 접근 가능 여부가 달라짐

## 왜 중요한가

Ansible 자동화가 아무리 재현 가능해도 **Controller가 Managed Node에 접근할 수 없으면 실행 자체가 불가능**하다.

따라서 Snapshot은 복구 수단이면서 동시에 SSH Bootstrap 상태를 되돌릴 수 있는 새로운 장애 요인이 된다.

## 설계된 대응

- 사용자별 SSH Key 존재 여부 확인
- 사용자 × VM 조합별 SSH 접속 테스트
- 정상 조합은 SKIP
- 실패한 조합만 공개키 재배포
- Host Key 변경은 무조건 무시하지 않음
- Root Password를 Git에 저장하지 않음

## 현재 상태

해당 복구 자동화 Issue는 **Open**이다. 따라서 “해결 완료”로 쓰지 않는다.

## 근거

- Issue #27: https://github.com/seokpan/seokpan-infra/issues/27

---
