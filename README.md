# SeokPan Project Documentation

이 저장소는 **석판팀 「石나가는 판단」 1차 프로젝트의 공용 문서 저장소**입니다.

프로젝트 기획·설계 기준 문서, 구현 단계의 공용 기준, 변경 이력 및 주요 아키텍처 자료를 관리합니다.

## Planning & Design

01~08 문서는 1차 프로젝트의 기획·설계 기준입니다.

| 순서 | 문서 |
| --- | --- |
| 01 | [서비스 요구사항 및 기능 명세 통합](01_SeokPan_서비스_요구사항_및_기능_명세_통합.pdf) |
| 02 | [핵심 문제 및 검증 목표](02_SeokPan_핵심_문제_및_검증_목표.pdf) |
| 03 | [논리 역할 및 서비스 목록](03_SeokPan_논리_역할_및_서비스_목록.pdf) |
| 04 | [기술 비교 및 논리 아키텍처](04_SeokPan_기술_비교_및_논리_아키텍처.pdf) |
| 05 | [물리 아키텍처](05_SeokPan_물리_아키텍처.pdf) |
| 06 | [Ansible 자동화·테스트 설계](06_SeokPan_Ansible_자동화_테스트_설계.pdf) |
| 07 | [확장 호환형 MVP 도출](07_SeokPan_확장_호환형_MVP_도출.pdf) |
| 08 | [1차 프로젝트 기획안](08_SeokPan_1차_프로젝트_기획안.pdf) |

## Implementation Baseline

PDF 작성 이후 확정된 변경과 구현 단계의 공용 기준은 아래 문서를 함께 적용합니다.

| 문서 | 역할 |
| --- | --- |
| [PROJECT_CHANGES.md](PROJECT_CHANGES.md) | PDF 이후 확정된 변경·추가·삭제 결정의 이력 |
| [MVP_IMPLEMENTATION_BASELINE.md](MVP_IMPLEMENTATION_BASELINE.md) | 현재 App·Infra·GitOps가 함께 소비하는 MVP 구현 기준과 검증 상태 |

요구사항 범위는 07의 MVP를 우선합니다. 변경 이력은 명시된 항목에만 적용하며,
구현 기준 문서는 원 기획·설계의 범위를 임의로 확대하거나 대체하지 않습니다.
실제 코드·Manifest·자동화와 작업 상태는 각 구현 Repository에서 관리합니다.

## Historical Record → Current State Traceability

멘토링 Baseline, Troubleshooting, 회고·검증 기록처럼 **특정 시점의 사실을 보존하는 역사성 문서**는 이후 상태가 바뀌더라도 당시 내용을 현재 상태로 덮어쓰지 않습니다.

대신 후속 결정이나 운영 기준 변경이 실제로 발생한 경우 다음 흐름으로 추적할 수 있게 관리합니다.

```text
Historical record
→ 당시 사실 보존
→ 후속 결정/변경 근거
→ 현재 운영 기준
```

- 당시의 사실·수행 결과·판단은 그대로 보존합니다.
- 현재 기준이 달라진 경우에만 `후속 상태`, `현재 기준`, 관련 Issue/PR 등의 연결을 추가합니다.
- 현재 운영 기준은 우선 [PROJECT_CHANGES.md](PROJECT_CHANGES.md)의 해당 변경·결정과 실제 구현 Repository의 Issue/PR을 함께 확인합니다.
- 단순히 오래된 문서라는 이유만으로 현재 상태 안내를 일괄 추가하지 않습니다.
- Baseline은 기준 시점을 보존하고 이후 변화는 별도 상태 기록이나 후속 링크로 연결합니다.
- Troubleshooting은 사건 당시 원인·조치·검증을 보존하고, 이후 동일 영역의 운영 기준이 바뀐 경우에만 현행 기준으로 이어지는 링크를 추가합니다.

## Troubleshooting

프로젝트 구현 과정에서 발생한 문제 가운데 **원인·조치·재검증까지 완료된 사례**를 사례별 Markdown 보고서로 관리합니다.

- [트러블슈팅 보고서 목차](troubleshooting/README.md)
- 각 사례는 발생/발견 시기, 담당 역할, 영향 범위, 원인, 조치, 검증 결과와 GitHub 근거를 포함합니다.
- 진행 중인 문제는 각 구현 Repository의 Issue/Pull Request에서 추적하며, 해결이 검증된 뒤 트러블슈팅 보고서로 게시합니다.

## Mentoring

멘토링에서 받은 프로젝트 리뷰와 실제 반영 과정을 회차별로 기록합니다.

- [멘토링 기록 목차](mentoring/README.md)
- 1차 멘토링: 2026-09-01, 레드햇 김명환 멘토님
- 멘토 의견, 해당 시점의 GitHub 실제 상태, 다음 멘토링까지의 반영 추적과 변화 기록을 함께 관리합니다.
- 단순 회의록이 아니라 `피드백 → 판단 → 구현/조치 → 검증 → 변화`를 연결해 팀 공유 및 포트폴리오 근거로 활용합니다.

## Architecture

주요 아키텍처 이미지는 별도 디렉터리에서 관리합니다.

```text
seokpan-docs/
├─ logical-architecture/
└─ physical-architecture/
```

* `logical-architecture/`  : 서비스 흐름, Kubernetes 및 주요 구성요소의 논리 구조
* `physical-architecture/` : VM 배치, 네트워크 및 인프라 물리 구조
