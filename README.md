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

## Troubleshooting

프로젝트 구현 과정에서 실제로 발생하거나 검증 과정에서 발견된 문제는 사례별 Markdown 보고서로 관리합니다.

- [트러블슈팅 보고서 목차](troubleshooting/README.md)
- 각 사례는 발생/발견 시기, 현재 상태, 담당 역할, 영향 범위, 원인, 조치, 검증 결과와 GitHub 근거를 포함합니다.
- 프로젝트 종료까지 새 사례와 후속 검증을 동일 디렉토리에 계속 누적합니다.

## Architecture

주요 아키텍처 이미지는 별도 디렉터리에서 관리합니다.

```text
seokpan-docs/
├─ logical-architecture/
└─ physical-architecture/
```

* `logical-architecture/`  : 서비스 흐름, Kubernetes 및 주요 구성요소의 논리 구조
* `physical-architecture/` : VM 배치, 네트워크 및 인프라 물리 구조
