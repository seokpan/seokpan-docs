[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-002 — Harbor 설치 자동화가 TLS 인증서 부재에서 중단된 뒤 내부 CA/TLS 체계로 연결

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-27 |
| **상태** | **해결** |
| **주 담당** | **최유준 — CI/CD 및 모니터링·관측** |
| **영향 범위** | 정태훈(Kubernetes 플랫폼·애플리케이션 이미지 사용), 이유빈(Host/DNS), CI/CD 전체 |

## 문제 개요

초기 Harbor Ansible Role은 Docker 설치부터 Harbor 설치까지 자동화했지만, 실제 실행에서 TLS 인증서가 준비되지 않아 다음 단계로 진행할 수 없었다.

이 중단 자체는 잘못된 동작이 아니었다. 인증서가 없는데 HTTP 또는 임시 인증서로 억지 진행하지 않고 **assert에서 안전하게 중단**하도록 작성되어 있었다.

## 원인 분석

Harbor 설치 자동화와 별개로, 설치에 필요한 다음 선행 구성요소가 아직 연결되지 않았다.

```text
Internal CA
    ↓
Harbor Server Certificate
    ↓
Harbor Ansible Role
    ↓
Jenkins / BuildKit / Kubernetes의 이미지 사용 경로
```

즉 Harbor Role 자체보다 **CA 발급·배포 경로가 선행조건으로 빠져 있던 통합 문제**였다.

## 조치

후속 작업에서:

- `internal_ca` Role 추가
- 최상위 인증기관 인증서(Root CA) 최초 생성
- `tls_deploy` Role 추가
- Harbor용 인증서 발급
- DNS + IP SAN 포함
- Harbor FQDN을 기존 `harbor.stone.test`에서
  `harbor.seokpan.soldesk.store`로 정리
- CA Private Key는 Controller 담당자 보호 영역에 `0600`으로 관리하고 Git 제외
- 공개 CA 인증서는 이후 Harbor를 사용하는 구성요소의 신뢰 설정용으로 분리

## 검증

- Internal CA 최초 생성
- 재실행 `changed=0`
- Harbor 인증서가 Internal CA로 서명됐는지 확인
- SAN 포함 확인
- Harbor `install.sh` 전체 통과
- Harbor Admin UI 로그인 성공
- 이후 공용 hosts 자동화에서 모든 관리 VM이 공식 FQDN을 해석하도록 정합화

## 중요한 역할 경계

이 사례에서 **CA Private Key를 보호하는 것**과 **Kubernetes 관리자 kubeconfig를 특정 담당자 개인 Home에 귀속하는 것**은 서로 다른 문제다. 이 구분은 2026-09-01 TS-018에서 다시 중요하게 등장한다.

## 관련 근거

- Harbor 초기 자동화 PR #21: https://github.com/seokpan/seokpan-infra/pull/21
- Internal CA/TLS PR #34: https://github.com/seokpan/seokpan-infra/pull/34
- Harbor FQDN 정합화 PR #68: https://github.com/seokpan/seokpan-infra/pull/68
