[← 전체 트러블슈팅](../README.md) · [네트워크·Ansible 자동화 목차](README.md)

# NET-007 — VRouter node_exporter 방화벽 Zone 불일치

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-17 |
| **상태** | **해결** |
| **원인 조사** | **이유빈(ggbun2) — tcpdump / firewalld 기반 Network Root Cause 조사** |
| **구현** | **최유준(cyj200115-prog) — `vrouter_firewall` Role 수정** |
| **영향 범위** | VRouter-02~04의 `node_exporter` 9100/tcp 접근 및 Prometheus `node-exporter-external` 수집 경로 |

## 문제

Prometheus에서 VRouter-02, VRouter-03, VRouter-04의
`node_exporter` Target이 Down 상태로 확인됐다.

표면적으로는 다음과 같은 문제를 의심할 수 있었다.

```text
Prometheus 설정 문제
node_exporter 프로세스 문제
9100/tcp 서비스 문제
Network / Firewall 차단
```

조사 결과 `node_exporter` 자체 장애가 아니라,
worker에서 VRouter 관리 IP의 `9100/tcp`로 접근하는 트래픽이
firewalld의 허용 규칙과 다른 Zone으로 유입되면서 차단되는 문제였다.

## 원인

기존 VRouter 방화벽 정책에서는
`node_exporter`가 사용하는 `9100/tcp` 허용 규칙이
`internal` Zone에만 적용돼 있었다.

하지만 실제 worker에서 VRouter 관리 IP로 전달되는 트래픽은
VRouter의 `ens160` 인터페이스를 통해 유입됐고,
해당 인터페이스는 `external` Zone에 연결돼 있었다.

실제 흐름은 다음과 같았다.

```text
Worker
  ↓
VRouter 관리 IP:9100
  ↓
ens160
  ↓
external zone
  ↓
9100/tcp 허용 규칙 없음
  ↓
firewalld 차단
  ↓
Prometheus Target Down
```

즉 기존 정책은 다음과 같은 상태였다.

```text
internal zone
└─ node_exporter 9100/tcp 허용

external zone
└─ worker → VRouter 9100/tcp 트래픽 실제 유입
   └─ 해당 허용 규칙 없음
```

따라서 서비스가 정상 실행 중이어도
실제 패킷이 들어오는 Zone에서 접근이 허용되지 않아
Prometheus가 VRouter의 `/metrics`를 수집할 수 없었다.

## 조사

원인 조사에서는 Prometheus의 Target Down 상태만 보고
Exporter 장애로 단정하지 않고
실제 패킷 경로와 firewalld Zone을 확인했다.

tcpdump와 firewalld 상태를 기준으로 확인한 결과,
worker에서 VRouter 관리 IP로 전달되는 요청이
`ens160`을 통해 들어오는 것을 확인했다.

실제 원인 조사에서는 worker-01(`192.168.51.30`)에서
vrouter-02 관리 IP(`192.168.52.10`):`9100`으로 접근할 때
패킷이 `ens160`으로 유입되는 것을 tcpdump로 확인했다.

`ens160`은 `external` Zone에 속해 있었고,
해당 요청에 대해 VRouter가 ICMP `admin prohibited`를 반환하며
연결을 거부하는 것도 확인됐다.

```text
worker-01 (192.168.51.30)
        ↓
vrouter-02 (192.168.52.10:9100)
        ↓
ens160
        ↓
external zone
        ↓
기존 9100/tcp 허용 규칙 없음
        ↓
ICMP admin prohibited
```

따라서 기존 `internal` Zone의 `9100/tcp` 허용 규칙은
실제 worker 요청에 적용되지 않았다.

이를 통해 다음을 구분했다.

```text
node_exporter 실행 장애
        ≠
firewalld Zone 불일치에 의한 접근 차단
```

이번 문제의 Root Cause는 후자였다.

## 조치

별도의 일회성 Fix Playbook을 유지하지 않고,
기존 `vrouter_firewall` Role에 정책을 통합했다.

`external` Zone 전체에 `9100/tcp`를 무제한으로 개방하지 않고,
Prometheus 수집이 필요한 worker 대역만 최소 범위로 허용했다.

허용 대상 worker CIDR:

```text
192.168.51.0/24
192.168.52.0/24
```

허용 대상 Port:

```text
9100/tcp
```

정책의 목적은 다음과 같다.

```text
external zone 전체
        ↓
모든 출발지에 9100 허용
        X

필요한 worker CIDR
        ↓
VRouter 9100/tcp
        ↓
최소 범위 허용
        O
```

즉 실제 패킷 유입 Zone에 맞춰 규칙을 적용하면서도,
접근 범위는 필요한 worker 네트워크로 제한했다.

## 검증

### 1. Worker에서 VRouter `/metrics` 접근

worker에서 VRouter의 node_exporter `/metrics` endpoint에 접근한 결과
정상적인 HTTP 응답을 확인했다.

```text
HTTP 200
```

이를 통해 worker에서 VRouter의 `9100/tcp`까지
실제 통신이 가능해졌음을 확인했다.

### 2. Prometheus Target 검증

Prometheus의 `node-exporter-external` Target 상태를 확인한 결과:

```text
7/7 UP
```

을 확인했다.

따라서 개별 HTTP 연결뿐 아니라
실제 Prometheus 수집 경로에서도 정상적으로 Metrics가 수집되는 것을 확인했다.

### 3. Ansible 재실행 검증

정책을 `vrouter_firewall` Role에 반영한 뒤
동일한 상태에서 Role을 다시 실행했다.

재실행 결과 불필요한 추가 변경이 발생하지 않는 것을 확인했다.

```text
changed=false
```

따라서 해당 방화벽 규칙이 반복 실행 시에도
불필요하게 변경되지 않는 것을 검증했다.

## 결과

이번 문제는 `node_exporter` 프로세스 자체의 장애가 아니라,
실제 worker 트래픽이 유입되는 firewalld Zone과
기존 `9100/tcp` 허용 Zone이 달라 발생한 Network/Firewall 문제였다.

조치 후 다음을 확인했다.

- worker → VRouter `/metrics` HTTP 200
- Prometheus `node-exporter-external` Target 7/7 UP
- `vrouter_firewall` Role 재실행 시 `changed=false`
- `9100/tcp`를 external Zone 전체에 무제한 개방하지 않음
- 필요한 worker CIDR만 최소 범위로 허용

## 담당 범위

이번 문제의 원인 조사와 실제 구현 담당을 구분한다.

### 원인 조사

이유빈(ggbun2)이 Infra Issue #205에서
tcpdump와 firewalld 상태를 기반으로
실제 패킷 유입 경로와 Zone 불일치를 조사했다.

### 구현

최유준(cyj200115-prog)이 Infra PR #206에서
`vrouter_firewall` Role을 수정해
external Zone에 필요한 worker 대역의
`9100/tcp` 접근 정책을 반영했다.

따라서 원인 조사와 구현을 동일 담당자의 작업으로 표현하지 않는다.

## 판단 기준 및 주의사항

Prometheus Target이 Down이라고 해서
바로 Prometheus 또는 Exporter 자체 장애로 판단해서는 안 된다.

다음 경로를 함께 확인해야 한다.

```text
Prometheus
    ↓
출발지 Worker
    ↓
Network Route
    ↓
VRouter Interface
    ↓
firewalld Zone
    ↓
Port 허용 정책
    ↓
node_exporter
```

특히 firewalld를 사용하는 환경에서는
단순히 특정 Port의 허용 여부만 확인하는 것이 아니라,
**실제 트래픽이 어느 인터페이스와 Zone으로 유입되는지**를 함께 확인해야 한다.

또한 이번 조치는 다음과 같이 해석하지 않는다.

```text
external Zone에서 9100/tcp 전체 개방
```

실제 적용 기준은:

```text
필요한 worker CIDR
→ external Zone
→ 9100/tcp 최소 허용
```

이다.

이 사례의 본문은 Network Troubleshooting에 한 번만 유지하며,
CI/CD·Observability 영역에서는 별도 중복 Troubleshooting을 생성하지 않고
관련 Network 사례로 연결한다.

## 관련 근거

- Infra Issue #205 — VRouter-02/03/04 node_exporter Target Down: https://github.com/seokpan/seokpan-infra/issues/205
- Infra Issue #205 — ggbun2 tcpdump/firewalld 원인 조사 및 ICMP `admin prohibited` 확인: https://github.com/seokpan/seokpan-infra/issues/205#issuecomment-5709689567
- Infra PR #206 — external Zone에 node_exporter 9100/tcp worker 대역 최소 허용: https://github.com/seokpan/seokpan-infra/pull/206
- Infra PR #206 — 허용 CIDR·external Zone·재현성 검토: https://github.com/seokpan/seokpan-infra/pull/206#issuecomment-5709937802
- Infra PR #206 — `vrouter_firewall` 적용 후 Prometheus UP 유지 확인: https://github.com/seokpan/seokpan-infra/pull/206#issuecomment-5710275503
- Docs Issue #141 — VRouter node_exporter 방화벽 Zone 불일치 문서화: https://github.com/seokpan/seokpan-docs/issues/141
