[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-002 — Harbor 설치 자동화가 TLS 인증서 부재에서 중단된 뒤 내부 CA/TLS 체계로 연결

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-08-28 |
| **상태** | **해결** |
| **주 담당** | **최유준 — CI/CD 및 모니터링·관측** |
| **영향 범위** | Jenkins/BuildKit, Kubernetes 이미지 사용 경로 |

## 문제 개요

Harbor 자동화 실행 과정에서 HTTPS에 필요한 인증서가 준비되지 않아 설치가 안전하게 중단됐다.

Harbor는 Jenkins/BuildKit이 이미지를 Push하고 Kubernetes가 이미지를 Pull하는 공용 Registry이므로, 임시 HTTP나 자체서명 인증서를 무시하는 방식으로 넘어가면 이후 CI/CD 전체에서 TLS 예외 처리가 확산될 수 있었다.

## 원인 분석

Harbor 설치 자체보다 앞서 다음 항목이 확정돼야 했다.

- Harbor 공식 FQDN
- 내부 CA
- Harbor Server Certificate
- 인증서 배포 위치
- Harbor를 사용하는 클라이언트의 CA Trust 방식

즉 설치 자동화가 실패한 원인은 단순 패키지 문제라기보다 **TLS 선행조건이 아직 충족되지 않은 상태에서 HTTPS 기반 Harbor를 구성하려 했기 때문**이었다.

## 조치

프로젝트 Harbor 주소를 다음으로 확정했다.

```text
harbor.seokpan.soldesk.store
```

그리고 내부 CA를 기준으로 Harbor Server Certificate를 발급하고 Harbor 설정에 연결했다.

자동화에서는 인증서가 없을 때 임의로 설치를 계속하지 않고 실패시키며, 인증서가 준비된 뒤에만 Harbor HTTPS 구성을 진행하도록 했다.

## 검증

Harbor 설치와 TLS 적용 이후 다음을 확인했다.

```text
https://harbor.seokpan.soldesk.store
→ HTTP 200
```

또한 Harbor 인증서의 Issuer와 프로젝트 내부 CA의 Subject가 일치하는지 확인했고, CA 파일을 명시한 TLS 검증에서 정상적으로 신뢰 관계가 성립했다.

## Before → After

```text
Before
Harbor 설치 시도
→ TLS 인증서 없음
→ 자동화 중단

After
FQDN 확정
→ 내부 CA/Server Certificate 준비
→ Harbor HTTPS 구성
→ HTTPS 200 및 CA 신뢰 관계 확인
```

## 관련 근거

- Harbor 설치 자동화 PR #21: https://github.com/seokpan/seokpan-infra/pull/21
- 내부 CA/TLS 관련 PR #34: https://github.com/seokpan/seokpan-infra/pull/34
- Harbor 후속 자동화 PR #68: https://github.com/seokpan/seokpan-infra/pull/68
