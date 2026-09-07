[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-023 — CoreDNS 관리 Block newline escaping 결함으로 신규 Replica가 CrashLoopBackOff

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경·영향·원인·조치·검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-04 |
| **상태** | **해결** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | CoreDNS Rollout, Cluster DNS 가용성, Project Endpoint 자동화 |

## 문제 개요

TS-021의 Project Endpoint/CoreDNS 자동화를 최초 적용하는 과정에서, Ansible이 CoreDNS 관리 `hosts` Block을 삽입한 뒤 Render된 Corefile의 줄바꿈 구조가 깨졌다.

문제 Render의 핵심 형태는 다음과 같았다.

```text
# END ANSIBLE MANAGED PROJECT ENDPOINTS    forward . /etc/resolv.conf {
    max_concurrent 1000
}
```

`forward` 행이 관리 Block 종료 주석과 같은 줄에 붙으면서 주석 처리됐고, 다음 줄의 `max_concurrent`가 Corefile 최상위 Directive처럼 해석됐다.

신규 CoreDNS Replica는 기동하지 못하고 `CrashLoopBackOff`가 됐다.

CoreDNS Log:

```text
/etc/coredns/Corefile:22 - Error during parsing: Unknown directive 'max_concurrent'
```

## 사건 경계

이 장애는 TS-021의 원래 `SERVFAIL` 원인과 다르다.

```text
TS-021
Pod가 Project FQDN을 CoreDNS로 조회
→ upstream에 Record 없음
→ SERVFAIL

TS-023
그 SERVFAIL 해결을 위해 CoreDNS 관리 Block 자동화 적용
→ newline Render 결함
→ Corefile Parsing 실패
→ 신규 CoreDNS Replica CrashLoopBackOff
```

따라서 같은 Infra #99 / PR #108 흐름 안에서 발생했더라도 독립 Root Cause로 관리한다.

## 원인 분석

문제의 핵심은 CoreDNS 기능이나 Endpoint 값 자체가 아니라 **Ansible이 기존 Corefile과 관리 Block을 합치는 과정의 newline escaping/line boundary 처리**였다.

기존 Corefile의 다음 Directive와 관리 Marker 사이 경계가 명확한 줄 단위 구조로 보존되지 않아 `forward`가 주석에 흡수됐다.

이 사건은 단순한 Markdown/문자 표현 오류가 아니라 실제 CoreDNS Config Parser가 잘못된 구조를 읽어 Runtime Controller Replica가 기동하지 못한 구성 Render 결함이었다.

## 탐지

자동화는 CoreDNS 변경 후 Rollout 상태를 대기하고 있었다.

신규 Replica가 정상화되지 않으면서 약 120초 Rollout Gate가 Timeout됐고, 이후 CoreDNS Pod Log를 확인해 Parsing Error를 찾았다.

즉 변경 성공 여부를 ConfigMap Apply 반환값만으로 판단하지 않고 **실제 Rollout Gate까지 확인했기 때문에 장애를 감지**할 수 있었다.

## 복구

### 1. 변경 전 Backup 사용

CoreDNS 수정 전에 저장해 둔 기존 Corefile Backup으로 즉시 복구했다.

```text
잘못 Render된 Corefile
→ Rollout 실패
→ 변경 전 Backup 복원
→ CoreDNS 정상 상태 회복
```

### 2. Render/Preflight 보완

재적용 전에 관리 Block과 기존 Corefile의 경계를 줄 단위로 다시 처리하고, 실제 Patch 전에 Render 결과를 검사하도록 Preflight를 보완했다.

핵심은 단순히 특정 문자열 하나를 고친 것이 아니라:

- Marker 위치 확인
- 기존 Corefile 구조 보존
- 관리 Block의 시작/종료 Line 분리
- 다음 Directive와 newline 경계 확인
- 적용 전 Render 검증

을 Gate에 포함한 것이다.

### 3. 재적용

수정된 Render 결과로 CoreDNS 관리 Block을 다시 적용하고 Rollout을 재검증했다.

## 검증

복구·수정 후 다음을 확인했다.

### CoreDNS Runtime

- 신규 CoreDNS Pod 2개: Running
- Restart Count: 0
- Parsing Error 미재발

### DNS Regression

- Kubernetes Service DNS: PASS
- Redis Service DNS: PASS
- Harbor Project Endpoint DNS: PASS
- DB Project Endpoint DNS: PASS
- Game Project Endpoint DNS: PASS
- Grafana Project Endpoint DNS: PASS
- External DNS: PASS

### Ansible 재실행

동일 자동화를 다시 실행했을 때:

```text
CoreDNS change required = False
changed=0
failed=0
unreachable=0
```

으로 수렴했다.

따라서 복구만 하고 끝난 것이 아니라 수정된 Render/Preflight 경로로 다시 적용한 뒤 Runtime, DNS 회귀, 멱등성까지 확인했다.

## Before → Change → After

```text
Before
CoreDNS Project Endpoint 자동화 최초 적용
→ 관리 Block 종료 Marker와 기존 forward Directive의 newline 경계 손상
→ Corefile Parsing Error
→ 신규 CoreDNS Replica CrashLoopBackOff

Change
변경 전 Backup으로 rollback
→ line boundary 기반 Render 보완
→ Patch 전 Preflight 강화
→ 재적용 및 Rollout Gate 확인

After
CoreDNS Replica Running / restart 0
→ Service/Project/External DNS 회귀 PASS
→ 동일 Ansible 재실행 changed=0
```

## 운영상 반영한 교훈

Cluster-wide 설정 파일은 `apply 성공`만으로 완료 판정하지 않는다.

```text
Render
→ Preflight
→ Backup
→ Apply/Patch
→ Runtime Rollout
→ 기능 Regression
→ 재실행 Idempotency
```

순서로 Gate를 둔다.

이 기준은 CoreDNS처럼 한 줄의 구문 오류가 Cluster 전체 이름해석에 영향을 줄 수 있는 Shared Infra 변경에서 특히 중요하다.

## 관련 사건

- [TS-021 — Pod의 Project Endpoint가 CoreDNS에서 SERVFAIL로 실패](TS-021_CoreDNS_Project_Endpoint_SERVFAIL.md)

TS-021은 본 장애가 발생한 배경 작업이며, TS-023은 그 해결 자동화 구현 중 새로 발생한 독립 Render 장애다.

## 관련 근거

- Docs Issue #57: https://github.com/seokpan/seokpan-docs/issues/57
- Infra Issue #99: https://github.com/seokpan/seokpan-infra/issues/99
- Infra PR #108: https://github.com/seokpan/seokpan-infra/pull/108
- 선행 사건 TS-021: [TS-021_CoreDNS_Project_Endpoint_SERVFAIL.md](TS-021_CoreDNS_Project_Endpoint_SERVFAIL.md)
