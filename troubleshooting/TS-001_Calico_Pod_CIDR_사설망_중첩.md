[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-001 — Calico Pod CIDR 후보 설정이 프로젝트 사설망과 중첩

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 아래 내용만으로 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성했습니다.

| 항목 | 내용 |
|---|---|
| **시점** | 2026-08-27 |
| **상태** | **예방형 · 해결** |
| **주 담당** | **정태훈 — Kubernetes / Kubernetes 플랫폼** |
| **영향 역할** | 이유빈(Network), Kubernetes를 사용하는 전체 작업 |
| **핵심 범주** | Network / Calico / Configuration Drift |

## 발견 배경

Repository 안에 서로 다른 Calico Custom Resource 후보가 동시에 존재했다.

- 실제 프로젝트/Live Cluster 기준: `10.244.0.0/16`
- 다른 후보 파일: `192.168.0.0/16`

문제는 `192.168.0.0/16`이 프로젝트의 실제 사설망인 `192.168.51.0/24`~`192.168.54.0/24`를 포함한다는 점이었다.

## 실제 영향

이 시점의 **Live Cluster는 정상**이었다. 즉 실제 장애가 이미 발생한 사건은 아니다.

하지만 향후 Calico 설치 자동화에서 잘못된 후보 파일을 적용하면 Pod CIDR과 Underlay 사설망이 겹치면서 Routing 충돌을 만들 수 있었다. 따라서 이 사례는 **Runtime 장애가 아니라 Repository에 존재하던 잠재 설정 충돌을 사전에 제거한 사례**로 기록한다.

## 원인

- 같은 역할을 수행하는 후보 Custom Resource가 Repository에 공존
- 실제 Runtime 기준과 Repository 후보 기준이 단일화되지 않음
- 자동화가 본격화되기 전에 “어떤 파일이 공식 설정인가”가 코드 수준에서 제거되지 않음

## 조치

- `192.168.0.0/16`을 사용하는 위험 후보 삭제
- 프로젝트 기준 `10.244.0.0/16` Custom Resource 유지
- Live Cluster와 Repository 설정을 대조
- 다음 값도 함께 확인
  - BGP Disabled
  - MTU 1450
  - VXLAN CrossSubnet
  - NAT Enabled

## 검증

- Live Calico IPPool: `10.244.0.0/16`
- `Installation`: `Ready=True`, `Degraded=False`
- Node Route에서 `192.168.0.0/16` Pod Network 경로 없음
- 프로젝트 사설망과 Pod CIDR 비중복 확인
- YAML 파싱 및 `git diff --check` 통과

## Before → After

```text
Before
Repository에 10.244.0.0/16 / 192.168.0.0/16 후보 공존
        ↓
잘못된 후보가 자동화에 들어갈 가능성
        ↓
프로젝트 사설망과 Pod CIDR 충돌 위험

After
공식 10.244.0.0/16만 유지
        ↓
Live Runtime과 Repository 기준 일치
        ↓
Calico 설치 자동화의 입력 기준 단일화
```

## 역할 영향

- **정태훈:** Calico/Kubernetes 설정 단일화
- **이유빈:** Underlay 사설망과 Pod CIDR 간 충돌 방지
- **전체 팀:** 잘못된 CNI 자동화 적용 위험 제거

## 근거

- Issue: https://github.com/seokpan/seokpan-infra/issues/26
- PR #28: https://github.com/seokpan/seokpan-infra/pull/28

---
