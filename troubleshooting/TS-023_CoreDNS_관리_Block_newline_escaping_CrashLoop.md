[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-023 — CoreDNS 설정 줄바꿈 오류로 신규 Pod가 CrashLoopBackOff 발생

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경·영향·원인·조치·검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-04 |
| **상태** | **해결** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | CoreDNS Rollout, Kubernetes DNS 가용성, 프로젝트 도메인 자동화 |

## 문제 개요

TS-021의 프로젝트 도메인/CoreDNS 자동화를 처음 적용하는 과정에서 Ansible이 CoreDNS `hosts` 관리 Block을 삽입한 뒤, 생성된 Corefile의 줄바꿈 구조가 깨졌다.

문제가 발생한 Corefile의 핵심 형태는 다음과 같았다.

```text
# END ANSIBLE MANAGED PROJECT ENDPOINTS    forward . /etc/resolv.conf {
    max_concurrent 1000
}
```

`forward` 행이 관리 Block 종료 주석과 같은 줄에 붙으면서 함께 주석 처리됐고, 다음 줄의 `max_concurrent`가 Corefile 최상위 지시어처럼 해석됐다.

그 결과 신규 CoreDNS Pod는 기동하지 못하고 `CrashLoopBackOff`가 됐다.

CoreDNS Log:

```text
/etc/coredns/Corefile:22 - Error during parsing: Unknown directive 'max_concurrent'
```

## 사건 경계

이 장애는 TS-021의 원래 `SERVFAIL` 원인과 다르다.

```text
TS-021
Pod가 프로젝트 FQDN을 CoreDNS로 조회
→ 상위 DNS에 Record 없음
→ SERVFAIL

TS-023
SERVFAIL 해결을 위해 CoreDNS hosts Block 자동화 적용
→ 줄바꿈 처리 오류
→ Corefile Parsing 실패
→ 신규 CoreDNS Pod CrashLoopBackOff
```

따라서 같은 Infra #99 / PR #108 작업 중 발생했더라도 원인이 달라 별도 TS로 관리한다.

## 원인 분석

문제의 핵심은 CoreDNS 기능이나 Endpoint 값 자체가 아니라 **Ansible이 기존 Corefile과 관리 Block을 합치는 과정에서 줄바꿈 경계를 잘못 처리한 것**이었다.

기존 Corefile의 다음 지시어와 관리 구간 종료 표시 사이에 줄바꿈이 보존되지 않아 `forward`가 주석에 포함됐다.

이는 단순한 문서 줄바꿈 문제가 아니라 실제 CoreDNS Config Parser가 잘못된 구조를 읽어 신규 CoreDNS Pod가 기동하지 못한 설정 생성 오류였다.

## 탐지

자동화는 CoreDNS 변경 후 새 Pod가 정상적으로 기동하는지 기다리고 있었다.

신규 Pod가 정상화되지 않으면서 약 120초 뒤 Rollout 대기가 Timeout됐고, 이후 CoreDNS Pod Log를 확인해 Parsing Error를 찾았다.

즉 ConfigMap 적용 명령이 성공했다는 사실만으로 완료 처리하지 않고 **실제 CoreDNS Pod가 정상 기동하는지까지 확인했기 때문에 장애를 발견**할 수 있었다.

## 복구

### 1. 변경 전 Backup 사용

CoreDNS 수정 전에 저장해 둔 기존 Corefile Backup으로 즉시 복구했다.

```text
잘못 생성된 Corefile
→ Rollout 실패
→ 변경 전 Backup 복원
→ CoreDNS 정상 상태 회복
```

### 2. 적용 전 생성 결과 검증 보완

재적용 전에 관리 Block과 기존 Corefile의 경계를 줄 단위로 다시 처리하고, 실제 Patch 전에 생성될 Corefile 내용을 검사하도록 보완했다.

확인 항목:

- 관리 구간 표시 위치 확인
- 기존 Corefile 구조 보존
- 관리 Block 시작/종료 줄 분리
- 다음 지시어와 줄바꿈 경계 확인
- 적용 전 생성 결과 검증

### 3. 재적용

수정된 Corefile로 관리 Block을 다시 적용하고 새 CoreDNS Pod가 정상 기동하는지 재검증했다.

## 검증

복구·수정 후 다음을 확인했다.

### CoreDNS 상태

- 신규 CoreDNS Pod 2개: Running
- Restart Count: 0
- Parsing Error 미재발

### DNS 재검증

- Kubernetes Service DNS: PASS
- Redis Service DNS: PASS
- Harbor 프로젝트 도메인 DNS: PASS
- DB 프로젝트 도메인 DNS: PASS
- Game 프로젝트 도메인 DNS: PASS
- Grafana 프로젝트 도메인 DNS: PASS
- External DNS: PASS

### Ansible 재실행

동일 자동화를 다시 실행했을 때:

```text
CoreDNS change required = False
changed=0
failed=0
unreachable=0
```

으로 추가 변경 없이 끝났다.

따라서 기존 Corefile을 복구한 것에 그치지 않고, 수정된 자동화로 다시 적용한 뒤 CoreDNS Pod 상태, DNS 기능, Ansible 재실행 결과까지 확인했다.

## Before → Change → After

```text
Before
CoreDNS 프로젝트 도메인 자동화 최초 적용
→ 관리 Block 종료 주석과 기존 forward 지시어의 줄바꿈 경계 손상
→ Corefile Parsing Error
→ 신규 CoreDNS Pod CrashLoopBackOff

Change
변경 전 Backup으로 복구
→ 줄 단위로 Corefile 생성 방식 보완
→ 적용 전 생성 결과 검사
→ 재적용 후 CoreDNS Pod 기동 확인

After
CoreDNS Pod Running / restart 0
→ Service/Project/External DNS 모두 정상
→ 동일 Ansible 재실행 changed=0
```

## 운영상 반영한 교훈

CoreDNS처럼 한 줄의 구문 오류가 Kubernetes 전체 이름해석에 영향을 줄 수 있는 설정 파일은 적용 명령의 성공 여부만으로 완료 처리하지 않는다.

```text
설정 생성
→ 적용 전 내용 검증
→ 변경 전 Backup
→ Apply/Patch
→ Pod 정상 기동 확인
→ DNS 기능 재검증
→ Ansible 재실행 확인
```

순서로 확인한다.

## 관련 사건

- [TS-021 — Kubernetes Pod에서 프로젝트 도메인이 CoreDNS SERVFAIL로 조회 실패](TS-021_CoreDNS_Project_Endpoint_SERVFAIL.md)

TS-021은 본 장애가 발생한 배경 작업이며, TS-023은 그 해결 자동화 구현 중 새로 발생한 독립적인 CoreDNS 설정 생성 오류다.

## 관련 근거

- Docs Issue #57: https://github.com/seokpan/seokpan-docs/issues/57
- Infra Issue #99: https://github.com/seokpan/seokpan-infra/issues/99
- Infra PR #108: https://github.com/seokpan/seokpan-infra/pull/108
- 선행 사건 TS-021: [TS-021_CoreDNS_Project_Endpoint_SERVFAIL.md](TS-021_CoreDNS_Project_Endpoint_SERVFAIL.md)
