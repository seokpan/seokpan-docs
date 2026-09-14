# GitHub 협업 및 Repository 운영

## 1. 문서 목적과 적용 범위

이 문서는 「石나가는 판단」 1차 프로젝트에서 사용하는 GitHub Organization, Repository, Directory, Issue, Branch, Commit, Pull Request, Review, Merge, Evidence 및 문서 관리의 공통 운영 기준을 정의한다.

프로젝트는 Application Source, Kubernetes Desired State, Infrastructure Automation, Project Documentation을 동일한 변경 단위로 혼합하지 않고 책임과 변경 수명주기에 따라 분리하여 관리한다.

이 문서의 목적은 Git 명령어 사용법을 설명하는 것이 아니다. 다음 질문에 일관되게 답할 수 있는 운영 기준을 제공하는 것이 목적이다.

- 변경해야 할 자산은 어느 Repository에 두는가?
- Repository 안에서는 어느 Directory가 해당 자산을 소유하는가?
- 다른 Repository와 연결되는 변경은 어떻게 추적하는가?
- 어떤 상태에서 작업을 완료했다고 판단하는가?
- 계획, 구현, Merge, Runtime, 검증 결과를 어떻게 구분하는가?
- 프로젝트의 결정·변경·장애·검증 Evidence를 어떻게 보존하는가?

상세한 Kubernetes 구축·적용 명령은 11 Runbook, 공식 시험절차와 측정 결과는 12 검증·측정 계획, 역할별 실행·통합 구조와 Gate는 09 실시설계에서 다룬다.

---

## 2. 선행 설계와 현재 운영 기준

### 2.1 Baseline에서 정의된 책임 분리

[03 논리 역할 및 서비스 목록](./03_SeokPan_논리_역할_및_서비스_목록.md#section-2)은 기술이나 물리 배치보다 책임·상태 소유권·운영 책임을 먼저 분리한다.

[04 기술 비교 및 논리 아키텍처](./04_SeokPan_기술_비교_및_논리_아키텍처.md#section-10)는 CI/CD·GitOps와 Image Delivery를 독립된 책임으로 두고, [Infrastructure Automation](./04_SeokPan_기술_비교_및_논리_아키텍처.md#section-11) 역시 Application Runtime과 구분한다.

[06 Ansible 자동화·테스트 설계](./06_SeokPan_Ansible_자동화_테스트_설계.md#section-2)은 관리 책임 경계를 다음과 같이 구분한다.

```text
Ansible Bootstrap
→ Argo CD Desired State
→ Kubernetes Runtime Self-Healing

Jenkins Build/Test
→ Harbor Image
→ GitOps PR/Merge
→ Argo CD Sync
```

[07 확장 호환형 MVP](./07_SeokPan_확장_호환형_MVP_도출.md#section-16-2)는 후속 실행·협업 실시설계에서 `Repository·Directory`, Provider/Consumer Integration Contract, GitHub 협업 환경을 구체화하도록 인계한다.

따라서 현재 Repository 구조는 선행 설계의 책임 경계를 구현 단계의 파일 소유권과 변경 경계로 구체화한 구조로 본다.

단, 현재 구조의 모든 세부 Directory가 처음부터 정확히 같은 이유로 설계되었다고 소급해 단정하지 않는다. 이 문서는 Baseline의 책임 분리와 현재 실제 Repository 구조가 만드는 운영 경계를 기준으로 설명한다.

### 2.2 Current State 판단 기준

01~08은 기획·설계 Baseline이다.

PDF 이후의 변경은 `PROJECT_CHANGES.md`, 구현 공통 기준은 `MVP_IMPLEMENTATION_BASELINE.md`, 실제 코드·Manifest·자동화와 작업 상태는 각 구현 Repository를 우선 확인한다.

```text
Historical Baseline
→ Change / Decision
→ Current Repository
→ Issue / PR / Commit
→ Runtime Evidence
```

과거 문서에 특정 구조가 적혀 있다는 이유만으로 현재 구현 완료 상태로 판단하지 않는다.

---

## 3. Repository 구성과 Source of Truth

현재 주요 Repository는 다음 네 개다.

| Repository | 주요 책임 | Source of Truth |
| --- | --- | --- |
| `seokpan-infra` | On-premise Infrastructure 및 Ansible 자동화 | Host·VM 내부 설정·Network·Kubernetes Bootstrap·External Infrastructure 자동화 |
| `seokpan-gitops` | Kubernetes Desired State 및 Argo CD 배포 구성 | Cluster 위에서 지속적으로 유지할 Kubernetes 선언 상태 |
| `seokpan-app` | Frontend·Backend Application Source 및 Application Test | 서비스 구현·Application 계약·Application 자체 검증 |
| `seokpan-docs` | 공용 기획·설계·변경·검증·Troubleshooting 기록 | Historical Baseline과 결정·변경·검증 Evidence의 문서화 |

하나의 기능이 여러 Repository와 관계된다고 해서 동일한 설정을 여러 곳에서 중복 소유하지 않는다.

---

## 4. Repository를 분리한 이유

### 4.1 Application과 Infrastructure

`seokpan-app`은 서비스 기능과 Application 자체 동작을 관리한다.

`seokpan-infra`는 Application이 실행될 기반 환경을 구축한다.

따라서 Backend의 비즈니스 로직이나 Dockerfile과 VM Network·kubeadm Bootstrap을 같은 변경 단위로 관리하지 않는다.

이 경계는 Application 변경과 Infrastructure 변경의 Review 범위와 장애 영향 범위를 분리한다.

### 4.2 Infrastructure와 GitOps

두 Repository 모두 Kubernetes와 관계되지만 책임은 다르다.

| Repository | 핵심 질문 |
| --- | --- |
| `seokpan-infra` | Cluster와 외부 Infrastructure를 어떻게 구축·재구축할 것인가? |
| `seokpan-gitops` | 이미 존재하는 Cluster가 어떤 Kubernetes 상태를 계속 유지해야 하는가? |

Ansible은 Host·VM·Cluster Bootstrap과 외부 Provider를 관리하고, Argo CD 이후 Kubernetes Workload의 지속 Desired State는 GitOps가 소유한다.

### 4.3 Application과 GitOps

Application Source가 Merge되거나 Image가 생성되었다고 해서 서비스가 배포 완료된 것은 아니다.

```text
Application Source
→ Test / Build
→ Container Image
→ GitOps Desired State
→ Argo CD Sync
→ Kubernetes Runtime
→ Runtime Validation
```

각 단계는 별도로 추적한다.

Application Source와 Application Deployment State를 분리함으로써 Image 생성과 실제 Runtime 활성화를 동일 상태로 오해하지 않는다.

### 4.4 Documentation

`seokpan-docs`는 구현 자산의 복제본을 저장하는 Repository가 아니다.

문서 Repository는 다음의 Traceability를 담당한다.

```text
Historical Baseline
→ Decision / Change
→ Implementation
→ Validation
→ Current State
```

현재 Runtime 상태는 구현 Repository와 Evidence를 우선하며, 문서는 그 근거와 변화 과정을 연결한다.

---

## 5. Repository 내부 구조와 책임 경계

### 5.1 `seokpan-infra`

대표 구조는 다음과 같다.

```text
seokpan-infra/
├─ .github/
├─ ansible/
│  ├─ bootstrap/
│  ├─ inventory/
│  ├─ playbooks/
│  ├─ roles/
│  └─ tools/
└─ README.md
```

각 영역의 책임은 다음과 같이 해석한다.

| Directory | 책임 |
| --- | --- |
| `bootstrap/` | Ansible 실행환경 자체의 준비·선행조건 |
| `inventory/` | 관리 대상 Host와 Group 및 환경 입력 |
| `roles/` | 반복 가능한 구성 책임 단위 |
| `playbooks/` | Role과 Task의 실행 순서·조합 |
| `tools/` | 구축·검증·운영 보조 도구 |

Infrastructure 자동화에서는 “대상”, “재사용 가능한 설정 단위”, “실행 순서”를 분리한다.

Role 하나가 전체 인프라를 소유하거나 Playbook 안에 모든 설정을 직접 누적하는 구조를 피한다.

[06의 Ansible 프로젝트 구조](./06_SeokPan_Ansible_자동화_테스트_설계.md#section-6)를 Baseline으로 참고하되, 실제 구현 상태는 `seokpan-infra/main`을 우선한다.

### 5.2 `seokpan-gitops`

대표 구조는 다음과 같다.

```text
seokpan-gitops/
├─ .github/
├─ argocd/
│  └─ applications/
├─ apps/
├─ platform/
├─ cicd/
├─ observability/
└─ README.md
```

이 구조는 Kubernetes Runtime의 책임을 유형별로 분리한다.

| Directory | 책임 |
| --- | --- |
| `argocd/applications/` | Argo CD가 관리할 Desired State 경로 연결 |
| `apps/` | Frontend·Backend 등 서비스 Application Runtime |
| `platform/` | Gateway·Redis·Namespace/RBAC·Storage 등 공통 Runtime Platform |
| `cicd/` | Jenkins 등 CI/CD Runtime |
| `observability/` | Prometheus·Grafana·Loki·Alloy·Alertmanager 등 관측 Runtime |

`argocd/applications/`에는 Workload 자체보다 어떤 경로를 Argo CD가 관리할 것인지에 대한 Application 선언을 둔다.

실제 Workload는 성격에 맞는 `apps/`, `platform/`, `cicd/`, `observability/` 경로에서 관리한다.

따라서 Directory 경계는 단순한 파일 정리가 아니라 Runtime Responsibility와 Review Scope의 경계다.

### 5.3 `seokpan-app`

대표 구조는 다음과 같다.

```text
seokpan-app/
├─ .github/
├─ backend/
├─ frontend/
├─ docs/
├─ Jenkinsfile
├─ Jenkinsfile.image-pipeline
└─ README.md
```

Backend와 Frontend는 서로 다른 실행 Workload지만 하나의 서비스와 Application Integration Lifecycle을 공유하므로 하나의 Application Repository 안에서 관리한다.

반대로 Kubernetes Deployment Desired State는 Application Repository가 아니라 `seokpan-gitops`가 소유한다.

#### Backend

대표적으로 다음 자산을 함께 관리한다.

```text
backend/
├─ src/
├─ tests/
├─ migrations/
├─ scripts/
├─ docs/
├─ Dockerfile
├─ pyproject.toml
└─ uv.lock
```

구현 코드뿐 아니라 Migration, Test, 검증 Script, Container Build와 Dependency Lock을 Application Lifecycle의 일부로 본다.

#### Frontend

Frontend 역시 Source와 함께 Test/E2E, Container Build, Nginx 설정 및 Dependency Lock을 관리한다.

Application 자체의 재현 가능한 Build·Test에 필요한 자산은 Application Repository에 둔다.

### 5.4 `seokpan-docs`

현재 문서 Repository는 다음 역할의 자료를 분리한다.

```text
seokpan-docs/
├─ 01~08 Markdown Reference Mirror
├─ PROJECT_CHANGES.md
├─ MVP_IMPLEMENTATION_BASELINE.md
├─ troubleshooting/
├─ mentoring/
├─ logical-architecture/
├─ physical-architecture/
└─ baseline-pdf/
```

01~08 Markdown은 Baseline의 탐색·Anchor 참조용 Reference Mirror다.

`baseline-pdf/`는 해당 Baseline의 확정 당시 PDF Snapshot을 보존한다.

`PROJECT_CHANGES.md`는 Baseline 이후 확정된 변경을 기록하고, `MVP_IMPLEMENTATION_BASELINE.md`는 구현 Repository들이 함께 소비하는 공통 구현 기준과 검증 상태를 연결한다.

`troubleshooting/`은 해결과 재검증이 끝난 사건성 기록을 사례별로 보존한다.

`mentoring/`은 피드백 자체뿐 아니라 후속 판단·구현·검증의 변화를 추적한다.

#### 09~12 후속 문서 배치

09~12에서는 다음 구조를 사용한다.

```text
09/
  역할별 MVP 실행·통합 실시설계

10_GitHub_협업_및_Repository_운영.md

11/
  역할별 구축·자동화 Runbook

12/
  역할별 검증·측정 계획

baseline-pdf/
  기존 01~08 PDF Baseline
```

09·11·12는 역할별 문서가 여러 개이므로 Directory를 사용하고, 10은 팀 공통 운영 기준이므로 주요 문서 위치에 단일 파일로 둔다.

---

## 6. 자산 배치 판단 기준

새 파일이나 설정을 만들기 전에 “어디에 둘 것인가”를 먼저 판단한다.

| 변경 대상 | 기본 위치 |
| --- | --- |
| Host·VM·Network 설정 | `seokpan-infra` |
| kubeadm·Cluster Bootstrap | `seokpan-infra` |
| Ansible Inventory·Role·Playbook | `seokpan-infra/ansible` |
| Backend Source | `seokpan-app/backend` |
| Frontend Source | `seokpan-app/frontend` |
| Application Dockerfile | `seokpan-app`의 해당 Application |
| Application CI/Image Pipeline Logic | `seokpan-app` |
| Backend/Frontend Deployment·Service | `seokpan-gitops/apps` |
| Gateway·Redis·Namespace/RBAC | `seokpan-gitops/platform` |
| Jenkins Runtime/JCasC | `seokpan-gitops/cicd` |
| Prometheus·Grafana·Loki 등 | `seokpan-gitops/observability` |
| Argo CD Child Application | `seokpan-gitops/argocd/applications` |
| Baseline 이후 확정 결정 | `seokpan-docs/PROJECT_CHANGES.md` |
| 해결·재검증된 주요 장애 | `seokpan-docs/troubleshooting` |
| 실행설계·Runbook·Validation 문서 | `seokpan-docs`의 09·11·12 |

판단이 애매하면 파일 종류보다 **누가 그 Desired State를 소유하는가**를 우선한다.

---

## 7. Provider / Consumer와 Cross-Repository 관계

Repository는 독립적으로 존재하지만 프로젝트 Runtime은 서로 의존한다.

예를 들어 Backend Runtime 활성화는 다음과 같이 여러 Repository를 연결한다.

```text
seokpan-app
Source / Test / Container Contract
        │
        ▼
Artifact / Image
        │
        ▼
seokpan-gitops
Backend Kubernetes Desired State
        │
        ├──────────────┐
        ▼              ▼
seokpan-infra       Platform Runtime
DB/Network/CA      Redis/Gateway/RBAC
Provider               │
        └──────┬────────┘
               ▼
       Kubernetes Runtime
               │
               ▼
        Validation / Evidence
               │
               ▼
          seokpan-docs
```

Provider가 준비되었다고 Consumer Integration이 자동 완료되는 것은 아니다.

다음 상태를 별도로 기록한다.

```text
Provider Ready
≠ Consumer Wired
≠ Runtime Running
≠ Integration Validated
```

[07의 역할·Provider·Consumer](./07_SeokPan_확장_호환형_MVP_도출.md#section-11)를 상위 기준으로 사용한다.

---

## 8. GitHub 기본 작업 Lifecycle

1차 프로젝트는 `main` 중심 Feature/Task Branch 방식을 사용한다.

장기 `dev`, `staging` Branch는 사용하지 않는다.

기본 흐름은 다음과 같다.

```text
Task / Problem
→ Issue
→ Branch
→ Implementation
→ Local / Static / Runtime Validation
→ Commit
→ Pull Request
→ Review
→ Required Validation
→ Squash Merge
→ Runtime / Integration Verification
→ Issue Close
→ Branch Cleanup
```

Pure Documentation 변경처럼 Runtime 검증이 의미 없는 작업에서는 적용 가능한 단계만 수행한다.

Infrastructure·GitOps·Application Integration처럼 실제 실행 결과가 중요한 변경은 Merge만으로 완료 처리하지 않는다.

---

## 9. Issue 운영

작업은 가능한 한 Issue로 시작한다.

현재 `infra/app/gitops`에 적용된 작업 Issue Template은 다음 항목을 요구한다.

- 목적
- 작업 내용
- 완료 기준
- 검증
- 영향 및 주의사항
- 관련 자료

Issue는 작업 목록만 나열하는 공간이 아니라 “왜 필요한지”와 “무엇을 확인하면 끝나는지”를 먼저 고정하는 작업 계약으로 사용한다.

작업 중 범위가 변경되면 실제 의사결정이 추적 가능하도록 Comment 또는 후속 Issue/PR에서 기록한다.

다른 Repository에 선행조건이 있으면 해당 Issue를 명시적으로 연결한다.

---

## 10. Branch 운영

작업 Branch는 최신 `main`을 기준으로 만든다.

현재 팀 컨벤션의 기본 형식은 다음과 같다.

```text
<issue-number>_<domain>/<short-description>
```

Domain은 변경 대상의 책임 영역과 맞춘다.

예:

```text
infra
platform
apps
cicd
observability
docs
```

검증 후 폐기할 임시 Branch는 정식 Merge Branch와 구분할 수 있다.

임시 Branch에 여러 작업을 계속 누적해 사실상의 장기 `dev` Branch로 사용하지 않는다.

Merge가 완료된 작업 Branch는 후속 확인 후 정리한다.

---

## 11. Commit 운영

Commit은 하나의 논리적 변경 단위를 기준으로 한다.

서로 관계없는 기능, 설정, 문서 정리를 하나의 Commit에 불필요하게 섞지 않는다.

Commit Message는 단순 결과보다 변경 목적이나 범위를 식별할 수 있도록 작성한다.

최종 `main` 반영은 Squash Merge를 사용하므로 PR 제목과 설명은 main History에서 중요한 변경 기록이 된다.

개발 Branch의 세부 Commit은 작업·검증 과정의 이력으로 활용한다.

---

## 12. Pull Request와 Review

PR은 `main`을 대상으로 한다.

현재 공통 PR Template은 다음 항목을 요구한다.

- 목적
- 변경 내용
- 관련 Issue
- 검증
- 영향 및 주의사항
- 리뷰 요청사항

PR은 코드 전달 수단뿐 아니라 변경 근거와 검증 결과를 Review 가능한 형태로 묶는 Evidence다.

현재 확인된 GitOps 운영에서는 최소 1명의 Review와 Squash Merge를 사용한다.

Argo CD Application/Root처럼 여러 Runtime 경로에 영향을 줄 수 있는 변경은 Runtime Platform 관점의 Review를 우선 요청하는 팀 관례가 존재한다.

다만 Repository별 GitHub Settings에서 기술적으로 강제되는 규칙과 문서·관례로 운영되는 규칙은 구분한다.

현재 확인되지 않은 Branch Protection이나 Required Reviewer 설정을 임의로 공통 강제 규칙으로 확대하지 않는다.

---

## 13. `.github` Template의 역할과 현재 적용 범위

현재 `seokpan-infra`, `seokpan-app`, `seokpan-gitops`에는 Repository-local `.github/`의 Issue Template과 PR Template이 존재한다.

이를 통해 다음 항목을 사람의 기억에만 의존하지 않고 작업 입력 형식에 반영한다.

```text
Why
→ What
→ Done Criteria
→ Validation
→ Impact
→ Related Work
```

현재 `seokpan-docs`에는 동일한 Repository-local `.github/` Directory가 확인되지 않는다.

따라서 네 Repository 모두에서 동일 Template이 기술적으로 적용된다고 표현하지 않는다.

`seokpan-docs`의 실제 작업도 Issue·Branch·PR·Review 흐름을 사용하지만, Template 적용 범위는 현재 Repository 상태와 구분하여 기록한다.

---

## 14. Merge, Running, Validated의 구분

프로젝트에서는 다음 상태를 혼동하지 않는다.

| 상태 | 의미 |
| --- | --- |
| Planned | 필요성이나 후속 계획만 존재 |
| Defined | 계약·구조·절차가 정의됨 |
| Implemented | 코드·설정·자산이 작성됨 |
| Merged | 변경이 `main`에 반영됨 |
| Running | 실제 Runtime에서 실행 중 |
| Validated | 성공 조건에 따라 실제 검증 완료 |
| Partial | 일부 조건만 충족 |
| Blocked | 선행조건이나 장애로 다음 단계 진행 불가 |
| Not Tested | 구현 여부와 별개로 해당 시험 미실시 |

따라서:

```text
Merged ≠ Running
Running ≠ Validated
```

이다.

예를 들어 Deployment Manifest가 main에 존재해도 `replicas: 0`이면 Application Runtime 실행 완료가 아니다.

Secret 공급 자산이 준비되었더라도 실제 Consumer가 연결하지 않았다면 Provider Integration 완료가 아니다.

---

## 15. GitOps 변경과 Rollback

Argo CD가 관리하는 Kubernetes Resource의 지속 상태는 Git을 Source of Truth로 한다.

Sync 이후 문제가 발견되었을 때 장기적인 해결책으로 `kubectl edit`, `kubectl patch` 등 Runtime 수동 Drift를 유지하지 않는다.

기본 Rollback 흐름은 다음과 같다.

```text
Problem Detected
→ Git Revert 또는 후속 수정
→ Review
→ Merge
→ Argo CD Sync
→ Runtime Verification
```

장애 분석이나 긴급 확인 과정에서 Runtime 명령을 사용할 수는 있지만, 지속 상태 변경은 Git에 남겨야 한다.

---

## 16. Cross-Repository 작업 관리

하나의 목표가 여러 Repository를 사용하더라도 모든 변경을 한 Repository에 억지로 넣지 않는다.

각 Repository가 자신의 책임 자산만 변경하고 관련 Issue/PR을 서로 연결한다.

Cross-Repository 작업에서는 다음을 명시한다.

- Provider
- Consumer
- 제공 Contract
- 선행조건
- 관련 Issue/PR
- 실제 완료 조건
- Integration Validation 책임

한 Repository의 PR Merge를 전체 Integration 완료로 확대 해석하지 않는다.

---

## 17. Evidence 운영

프로젝트의 주요 주장은 가능한 경우 실제 Evidence로 연결한다.

| Evidence 종류 | 예 |
| --- | --- |
| Git | Issue, PR, Commit |
| Runtime | Node·Pod·Service·Endpoint 상태 |
| Test | PASS/FAIL 및 Test Result |
| CI/CD | Jenkins Run, Build Result |
| Artifact | Image Digest, SBOM, Scan |
| Data | Checksum, Replication, Snapshot |
| Recovery | 장애/복구 시각, RTO/RPO |
| Observability | Metric, Log, Alert |

“설치했다”, “안정적이다”, “복구된다”와 같은 설명만으로 검증 완료 처리하지 않는다.

[06의 Evidence 설계](./06_SeokPan_Ansible_자동화_테스트_설계.md#section-17) 및 [07의 핵심 검증과 Evidence 설계](./07_SeokPan_확장_호환형_MVP_도출.md#section-9)를 연결 기준으로 사용한다.

---

## 18. Documentation과 Historical Traceability

01~08은 프로젝트 기획·설계의 Historical Baseline이다.

기본 탐색과 Section 참조에는 Markdown Reference Mirror를 사용하며, 원본 PDF는 `baseline-pdf/`에 고정 Snapshot으로 보존한다.

Baseline 이후의 확정 변경은 `PROJECT_CHANGES.md`, 구현 공통 기준은 `MVP_IMPLEMENTATION_BASELINE.md`, 실제 구현과 Runtime 상태는 구현 Repository와 Evidence를 확인한다.

```text
01~08 Baseline
→ PROJECT_CHANGES
→ MVP_IMPLEMENTATION_BASELINE
→ Repository / Issue / PR / Commit
→ Runtime Evidence
→ 09~12
```

Historical Record는 이후 상태가 달라졌다고 해서 당시 내용을 현재 기준으로 덮어쓰지 않는다.

현재 기준이 달라진 경우 후속 결정·Issue·PR 또는 Current State 문서로 연결한다.

---

## 19. Troubleshooting 관리

진행 중인 문제는 해당 구현 Repository의 Issue/PR에서 추적한다.

문제가 해결되었다는 이유만으로 모든 Issue를 Troubleshooting 문서화하지 않는다.

다음과 같은 가치가 있는 사례를 우선 `seokpan-docs/troubleshooting/`에 승격한다.

- 원인 추적 과정이 재사용 가능함
- 여러 구성요소의 책임 경계를 보여줌
- 장애·복구 Evidence가 존재함
- 잘못된 운영 가정이나 자동화 결함을 수정함
- 후속 운영 기준에 영향을 줌

Troubleshooting 문서는 사건 당시의 사실·원인·조치·검증을 보존한다.

후속 운영 기준이 변경되었다면 원 기록을 수정하기보다 현재 기준으로 이어지는 연결을 추가한다.

---

## 20. Mentoring과 협업 기록

멘토링 기록은 단순 회의록으로 관리하지 않는다.

```text
Feedback
→ 판단
→ Issue / Task
→ 구현·조치
→ Validation
→ 변화
```

형태로 실제 프로젝트 변화와 연결한다.

GitHub는 실제 변경과 Task의 Source of Truth로 사용하고, 설명·가이드·회의·발표 자료와 같은 보조 지식은 별도 문서 도구를 사용할 수 있다.

동일한 Task 상태를 여러 도구에서 별도로 관리하여 서로 다른 Current State를 만들지 않는다.

---

## 21. Security와 Secret 관리

Password, Token, Private Key, 실제 Application Secret, kubeconfig Credential 등의 민감정보를 일반 Git 파일에 평문으로 저장하지 않는다.

Git에는 Secret 값보다 Contract를 관리한다.

```text
Secret Name
Key
Provider
Consumer
Injection Path
Validation Method
```

민감정보가 Git History에 유입된 경우 현재 파일에서 삭제하는 것만으로 해결됐다고 판단하지 않는다.

노출 범위, 과거 Commit, Credential 폐기·재발급 필요성을 별도로 확인한다.

---

## 22. AI·Codex 보조 작업 원칙

AI 개발 도구는 코드·문서·분석 초안을 생성하거나 수정하는 보조 도구로 사용할 수 있다.

AI가 생성했다는 사실은 구현 또는 검증 Evidence가 아니다.

AI 또는 Codex가 만든 변경도 다음 과정을 생략하지 않는다.

```text
Source 확인
→ Owner 검토
→ 실제 Diff 확인
→ Test / Validation
→ Review
→ Merge
```

Repository 실제 구조를 확인하지 않은 상태에서 대규모 파일·Directory를 임의로 생성하지 않는다.

기존 Issue·PR·코드와 충돌하는 경우 실제 Repository 상태를 우선한다.

[07의 AI 개발 도구와 ANALYSIS Runtime 경계](./07_SeokPan_확장_호환형_MVP_도출.md#section-3-2)를 따른다.

---

## 23. Baseline 계획과 현재 적용 상태의 구분

[07 실행·협업 실시설계 인계](./07_SeokPan_확장_호환형_MVP_도출.md#section-16-2)는 GitHub Organization·Repository·Project·Issue·PR·Review·CODEOWNERS 등을 후속 협업환경 구성 대상으로 제시했다.

그러나 Baseline에서 설정 대상으로 제시된 항목을 실제 현재 운영 완료 상태와 동일시하지 않는다.

현재 Repository에서 구현 또는 사용 Evidence가 확인된 항목만 현재 운영 기준으로 기록한다.

예를 들어 현재 Repository 파일에서 CODEOWNERS가 확인되지 않는다면 이를 활성화된 운영 규칙이라고 작성하지 않는다.

GitHub Project 역시 실제 Current State를 확인한 뒤 운영 현황을 판정한다.

---

## 24. 현재 구조에서 확인되는 운영 효과

현재 Repository와 작업 이력에서 다음 효과를 확인할 수 있다.

### 책임 경계

Application Source, Kubernetes Desired State, Infrastructure Automation, Documentation이 실제 별도 Repository에서 관리되고 있다.

### Review 범위 축소

Application 기능 변경과 Infrastructure 변경, GitOps Runtime 변경을 서로 다른 PR로 검토할 수 있다.

### Build와 Deploy 상태 분리

Application Pipeline과 Kubernetes Desired State가 분리되어 Image 생성과 Runtime 활성화를 독립적으로 관리할 수 있다.

### Provider / Consumer 추적

DB·Credential·CA 같은 Provider 자산과 Application Consumer 변경을 별도 Issue/PR로 추적할 수 있다.

### GitOps Rollback

Kubernetes의 지속 Desired State 변경을 Git History에서 Revert·Review할 수 있다.

### 재현성

Application Dependency Lock, Ansible Role/Playbook, Kubernetes Manifest와 CI Pipeline이 Repository에 버전 관리된다.

### Evidence 축적

Issue·PR·Commit·Troubleshooting·Runtime Evidence가 구현 결과뿐 아니라 판단과 검증 과정을 남긴다.

이 효과는 Repository를 많이 사용했기 때문에 발생하는 것이 아니라, 각 Repository와 Directory의 Source of Truth를 구분했기 때문에 얻어진다.

---

## 25. 작업 완료 기준

GitHub 작업의 기본 완료 흐름은 다음과 같다.

```text
Issue Scope 충족
→ Implementation 완료
→ 필요한 Validation 완료
→ Review 완료
→ Squash Merge
→ 필요한 Runtime / Integration Verification
→ Evidence 연결
→ Issue Close
→ Branch Cleanup
```

문서-only 변경과 Runtime 변경의 완료조건은 다를 수 있다.

Runtime 영향이 있는 변경은 Merge만으로 완료 처리하지 않는다.

작업이 여러 Repository에 걸쳐 있다면 모든 선행·후행 Integration Gate가 충족되기 전에는 개별 Repository 완료를 전체 프로젝트 완료로 표현하지 않는다.

---

## 26. 운영 원칙 요약

「石나가는 판단」 프로젝트의 GitHub 운영 목적은 Repository와 Branch 수를 늘리는 것이 아니다.

핵심은 다음 네 가지다.

1. **책임이 다른 자산의 Source of Truth를 분리한다.**
2. **모든 변경은 이유·영향·검증을 추적할 수 있게 한다.**
3. **Merge와 Runtime 성공, Validation 완료를 구분한다.**
4. **Baseline부터 Current State까지의 변화와 Evidence를 연결한다.**

```text
Responsibility
→ Repository / Directory
→ Issue
→ Branch / Change
→ Review
→ Merge
→ Runtime
→ Validation
→ Evidence
```

이 흐름을 통해 팀원이 “무엇을 어디에서 변경해야 하는가”, “누가 제공하고 누가 소비하는가”, “무엇을 확인해야 완료인가”를 동일한 기준으로 판단할 수 있도록 한다.
