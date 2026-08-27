# 石나가는 판단 Project Documentation

솔데스크 석판팀 **「石나가는 판단」** 프로젝트의 기획, 아키텍처 및 설계 문서를 관리하는 Repository입니다.

본 프로젝트는 실시간 투표 기반 오목 서비스를 대상으로 **On-premise 환경에서 Kubernetes 기반 서비스 인프라를 구축하고, Ansible을 이용한 자동화와 장애·복구·성능 검증을 수행하는 것**을 목표로 합니다.

## Project Documents

`docs/` 디렉터리에는 프로젝트의 공식 기획·설계 문서를 PDF 형태로 관리합니다.

| 순서 | 문서                 |
| -- | ------------------ |
| 01 | 서비스 요구사항·기능        |
| 02 | 핵심 문제·검증 목표        |
| 03 | 논리 역할·서비스 목록       |
| 04 | 기술 비교·논리 아키텍처      |
| 05 | 물리 아키텍처            |
| 06 | Ansible 자동화·테스트 설계 |
| 07 | 확장 호환형 MVP         |
| 08 | 최종 기획안             |

## Architecture

주요 아키텍처 이미지는 별도 디렉터리에서 관리합니다.

```text
docs/
├─ logical-architecture/
└─ physical-architecture/
```

* `logical-architecture/` : 서비스 흐름, Kubernetes 및 주요 구성요소의 논리 구조
* `physical-architecture/` : VM 배치, 네트워크 및 인프라 물리 구조

## Related Repositories

프로젝트 구현은 역할별 Repository로 분리하여 관리합니다.

* `seokpan/seokpan-infra` — On-premise Infrastructure 및 Ansible
* `seokpan/seokpan-app` — Frontend / Backend Application
* `seokpan/seokpan-gitops` — Kubernetes Application Desired State 및 Argo CD GitOps

본 Repository는 위 세 Repository에 공통으로 적용되는 프로젝트 기획·설계 문서의 기준 저장소입니다.
