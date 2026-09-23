# 09_MVP_실행·통합_실시설계

## Delivery / Observability

---

## 목차

1. [목적과 범위](#1-목적과-범위)
2. [선행 기준과 상태 판정](#2-선행-기준과-상태-판정)
3. [역할과 책임 경계](#3-역할과-책임-경계)
4. [Delivery Current State](#4-delivery-current-state)
5. [Observability Current State](#5-observability-current-state)
6. [Provider / Consumer Contract](#6-provider--consumer-contract)
7. [Integration 흐름](#7-integration-흐름)
8. [Integration Gate](#8-integration-gate)
9. [Gate 요약](#9-gate-요약)
10. [남은 Blocker와 Gap](#10-남은-blocker와-gap)
11. [Critical Path](#11-critical-path)
12. [완료 기준](#12-완료-기준)
13. [Traceability](#13-traceability)

---

# 1. 목적과 범위

## 1.1 목적

본 문서는 「石나가는 판단」 1차 프로젝트에서 **Delivery(Harbor·Jenkins·Argo CD·GitOps Promotion)** 와 **Observability(Prometheus·Grafana·Loki·Alloy·Alertmanager·외부 VM Exporter)** 영역의 책임, Current State, Provider/Consumer 관계, Integration Gate와 완료 경계를 실제 실행·검증 결과 기준으로 정리한다.

단순히 Jenkins나 Prometheus가 설치되어 있는지를 확인하는 것이 아니라, 다음 두 흐름이 실제로 끝까지 연결되는지를 판정 대상으로 삼는다.

**Delivery 흐름**

```text
seokpan-app main Commit
    ↓
Jenkins main Image Pipeline (P1 재검증)
    ↓
Rootless BuildKit Build + SBOM/Provenance
    ↓
Harbor Candidate Push → Trivy Scan → Health Smoke
    ↓
Harbor Final Tag / Digest 확정 + Evidence
    ↓
GitOps Promotion PR 자동 생성 (auto-merge 없음)
    ↓
Review / Merge
    ↓
GitHub Webhook → Argo CD Sync
    ↓
Kubernetes Pod가 검증된 Digest로 Running / Ready
```

**Observability 흐름**

```text
Kubernetes / Application / 외부 VM
    ↓
Metric (Prometheus Scrape) · Log (Alloy → Loki)
    ↓
Grafana 조회 (Provisioning 파일 기반 Dashboard / Datasource)
    ↓
Prometheus Rule 평가 → Alertmanager
    ↓
E-mail 통보 (firing → resolved)
    ↓
장애 Timeline / Evidence 재구성
```

07 문서는 이 영역을 **M4 Delivery/Observability**(Harbor·Jenkins·Argo CD·Metric·Log·E-mail 연결)로 정의했고, 06 문서는 **G-06 Delivery**(Commit→Digest→Healthy)와 **G-07 Observability**(Scrape·Log·Alert firing/resolved, Metric·Log 조회와 E-mail)를 Gate로 정의했다. 본 문서는 이 두 Gate가 실제 환경에서 어디까지 통과했는지를 기록한다.

- 상세 실행 명령·적용·복구·Rollback 절차: 11 구축·자동화 Runbook
- 공식 Test Case·측정·PASS/FAIL·Evidence 상세: 12 검증·측정 계획
- Repository·Issue·Branch·PR·Review Governance: [`10_GitHub_협업_및_Repository_운영.md`](../10_GitHub_협업_및_Repository_운영/10_GitHub_협업_및_Repository_운영.md)

---

## 1.2 범위

| 영역 | 주요 구성 | 검증 목적 |
| --- | --- | --- |
| Harbor | `harbor-01`(`192.168.53.61`), `harbor.seokpan.soldesk.store`, v2.15.2 | Private Registry, TLS, Robot Account 권한 분리, Tag Immutability |
| Jenkins | `cicd` namespace, Jenkins `2.568.2-lts-jdk21`, JCasC, Plugin Lock | CI Runtime 재현성, Credential 주입 경계 |
| Rootless BuildKit | `moby/buildkit:v0.32.2-rootless`(Digest 고정) | 권한 없는 Image Build/Push |
| CI Pipeline | `seokpan-app/Jenkinsfile`(PR), `Jenkinsfile.image-pipeline`(main) | Test → Build → Scan → Smoke → Digest → Evidence |
| GitOps Promotion | `scripts/promote_gitops.py`, `seokpan-ci-bot` 계정 | 검증 Digest의 GitOps PR 자동 생성 |
| Argo CD | v3.4.7, App-of-Apps, `selfHeal: true` | Desired State Sync, Self-Heal, Git Revert Rollback |
| Argo CD Webhook | HTTPRoute `/api/webhook` + ReferenceGrant + Webhook Secret | 변경 감지 지연 단축 |
| Prometheus | kube-prometheus-stack Chart `88.5.4`, `worker-01` Local PV | Cluster·Application·외부 VM Metric 수집 |
| Loki / Alloy | Loki `3.7.6`(`worker-02` Local PV), Alloy `v1.6.1` DaemonSet | Pod Log·Node Journal 수집 |
| Alertmanager | Resend SMTP(`smtp.resend.com:587`), 수신 `seokpan@soldesk.store` | Alert firing/resolved E-mail |
| Grafana | Sidecar Provisioning(Datasource/Dashboard ConfigMap) | Cluster 관점·서비스 관점 Dashboard 분리 |
| 외부 VM Exporter | node_exporter(VRouter 4·MaxScale·NFS·Ansible), mysqld_exporter, HAProxy 네이티브, Harbor | Kubernetes 밖 인프라 관측 |
| Observability NetworkPolicy | `observability/networkpolicy-observability.yaml` | Default Deny 전제의 최소 통신 허용 |

---

# 2. 선행 기준과 상태 판정

## 2.1 선행 기준

직접 상위 기준은 다음 문서다.

- [`02_SeokPan_핵심_문제_및_검증_목표.md`](../01-08_기획·설계_Baseline/02_SeokPan_핵심_문제_및_검증_목표.md) — C-06 관측·검증성
- [`06_SeokPan_Ansible_자동화_테스트_설계.md`](../01-08_기획·설계_Baseline/06_SeokPan_Ansible_자동화_테스트_설계.md) — 60 Delivery / 70 Platform 단계, G-06·G-07, F-10~F-13, Q-07, AUT-DEL·AUT-OBS
- [`07_SeokPan_확장_호환형_MVP_도출.md`](../01-08_기획·설계_Baseline/07_SeokPan_확장_호환형_MVP_도출.md) — M4, W-07·W-08, 13.1 Functional Freeze
- [`PROJECT_CHANGES.md`](../PROJECT_CHANGES.md) / [`MVP_IMPLEMENTATION_BASELINE.md`](../MVP_IMPLEMENTATION_BASELINE.md)
- [`mentoring/MENTORING-01_2026-09-01.md`](../mentoring/MENTORING-01_2026-09-01.md) — Dashboard 관점 분리, Alertmanager 외부 알림, Image 취약점 검사, 승인 절차

01~08은 Historical Baseline이며 당시 명칭(`harbor.stone.test`, `stone-judgment-*` 등)을 그대로 유지한다. 현행 기준 명칭은 `harbor.seokpan.soldesk.store`, `seokpan-*`이며(PROJECT_CHANGES 2026-08-31 Harbor Hostname 변경), 본 문서는 현행 명칭만 사용한다.

실제 상태는 `seokpan-gitops`·`seokpan-infra`·`seokpan-app`의 현재 `main`, Issue, PR, Runtime Evidence를 우선한다.

---

## 2.2 상태 표현 기준

| 상태 | 의미 |
| --- | --- |
| `Defined` | 설계 또는 구성 방법이 정의됨 |
| `Implemented` | 코드 또는 Manifest로 구현됨 |
| `Merged` | Git 저장소의 `main`에 반영됨 |
| `Running` | 실제 환경에서 현재 동작 중 |
| `Validated` | 실제 실행을 통해 정상 동작을 확인함 |
| `Partial` | 일부 조건만 검증됨 |
| `Not Tested` | 구현되어 있으나 실제 검증하지 않음 |
| `Not Implemented` | 구현 자체가 없음 |
| `Deferred` | MVP 범위에서 후순위(2차 프로젝트)로 보류 |

### 중요한 판정 원칙

```text
Implemented ≠ Merged ≠ Running ≠ Validated
```

Delivery 영역에서는 다음을 서로 다른 판정으로 분리한다.

```text
Image Build 성공
≠ Harbor Push 성공
≠ Scan / Smoke PASS
≠ Final Digest 확정
≠ GitOps PR 생성
≠ GitOps Merge
≠ Argo CD Synced
≠ Pod가 해당 Digest로 Ready
```

Observability 영역에서는 다음을 서로 다른 판정으로 분리한다.

```text
Observability Stack Running
≠ Target UP
≠ 실제 Metric Query 결과 존재
≠ Dashboard에 표시

Alertmanager Running
≠ Alert firing API 확인
≠ 실제 E-mail 수신
≠ resolved E-mail 수신
```

**추가 원칙**: ArgoCD가 `selfHeal: true`로 관리하는 리소스는 `kubectl edit/patch`로 고친 상태를 결과로 인정하지 않는다. Git 변경 → Review → Merge → Argo CD Sync를 거친 상태만 Current State로 기록한다(TS-035에서 실증된 원칙).

---

# 3. 역할과 책임 경계

## 3.1 주요 책임

Delivery / Observability 영역(담당: 최유준)은 **Application Source가 검증된 Image로 Runtime까지 도달하는 경로**와 **모든 구성요소의 상태를 Metric·Log·Alert로 재구성할 수 있는 검증 계층**을 담당한다.

- Harbor 설치·Project·Robot Account·Tag Immutability·GC 자동화(`harbor` Role)
- Harbor 내부 CA 신뢰 경로(BuildKit `SSL_CERT_FILE`, Node `ca_trust`)
- Jenkins Controller·JCasC·Plugin Lock·Agent PodTemplate(`cicd/`)
- Jenkins Credential 주입 경로(`jenkins_secrets` Role → K8s Secret → env → JCasC)
- CI Pipeline(PR 검증 / main Image Pipeline)과 검증 Evidence 형식
- GitOps Promotion 자동화(Jenkins → GitOps PR)
- Argo CD Bootstrap(`argocd_bootstrap` Role)과 Webhook 변경 감지
- kube-prometheus-stack·Loki·Alloy·Alertmanager·Grafana Desired State(`observability/`)
- Alertmanager SMTP Credential 주입(`alertmanager_secret`)과 Preflight(`alertmanager_smtp_preflight`)
- 외부 VM Exporter 수집 경로(EndpointSlice/ServiceMonitor, VRouter·LB 방화벽/설정 연동)
- Grafana Dashboard(Cluster·외부 VM·MariaDB·서비스 KPI)
- Observability NetworkPolicy

## 3.2 직접 소유하지 않는 영역

| 영역 | 주요 Owner |
| --- | --- |
| Physical Network·VRouter·LB·공통 Ansible 실행환경 | Network / Infra Automation(이유빈) |
| Kubernetes Cluster·Gateway·Redis·Backend/Frontend Desired State | Kubernetes & Application Integration(정태훈) |
| MariaDB·MaxScale·NFS·Backup/Restore, node_exporter_linux·mysqld_exporter Role | Data / Storage / Recovery(김상희) |
| Application `/metrics` 계측·Log 내용 | `seokpan-app`(정태훈) |
| Application Runtime Pull 전용 Robot·`harbor-pull-secret`(application ns) | Kubernetes & Application Integration(정태훈, Infra #180/PR #183) |

**경계 원칙**: 외부 VM에 Exporter를 설치하거나 방화벽을 여는 작업처럼 다른 담당 영역(VRouter·LB·DB 서버)을 건드리는 변경은 소유 담당자 확인 후 진행한다(단독 결정 지양). 실제로 VRouter external zone 규칙(Issue #205)과 lb01 HAProxy stats 활성화는 네트워크 담당 확인을 거쳐 진행했다.

---

# 4. Delivery Current State

## 4.1 Harbor Registry

| 항목 | 현재 값 |
| --- | --- |
| VM / 주소 | `harbor-01` / `192.168.53.61` |
| FQDN | `harbor.seokpan.soldesk.store`(SAN: DNS + IP 포함) |
| 버전 | v2.15.2(offline installer, `harbor_version` 고정) |
| Project | `seokpan`(Private) |
| 기동 | systemd Unit으로 VM 재부팅 후 자동 기동(TS-010) |
| GC | GC Policy 자동화(Infra PR #69) |
| Metric | Harbor 자체 `/metrics`(9090) → Prometheus `harbor-metrics` Job |

### Robot Account 권한 분리

| Robot Account | 용도 | Consumer |
| --- | --- | --- |
| `robot$seokpan+seokpan-ci` | Push/Pull(Image Build) | Jenkins BuildKit(`harbor-robot-account`) |
| `robot$seokpan+seokpan-api` | Harbor API(Guard/Promote/Digest 조회/Cleanup) | Jenkins python 컨테이너(`harbor-api-robot-account`) |
| Runtime pull-only Robot | Application Pod Image Pull 전용 | `application/harbor-pull-secret`(정태훈 소유) |

Admin Credential은 Pipeline에서 사용하지 않는다. Robot 실제 값은 Vault(`inventory/group_vars/all/vault.yml`)에만 존재하고, GitOps에는 Secret 이름 참조만 존재한다.

### Tag Immutability

```text
Final Tag  : git-<main-sha-12>        → immutable (덮어쓰기·삭제 거부)
Candidate  : scan-<main-sha-12>-<BUILD_NUMBER> → 예외 (Scan 실패 시 정리 가능)
```

Harbor Immutability Rule의 `tag_selectors`는 selector 1개만 유효하게 평가되므로 `excludes: "scan-**"` 단일 selector 방식으로 구현했다(TS-034). 동일 Tag 재Push 거부는 TS-027에서 실증됐다.

판정: `Validated`.

---

## 4.2 Harbor 내부 CA 신뢰 경로

Harbor TLS 인증서는 프로젝트 내부 Root CA로 서명되어 있으며, 인증서를 검증하는 주체가 셋으로 나뉘어 각각 별도로 신뢰 경로를 구성했다.

```text
Jenkins BuildKit Token 요청   → SSL_CERT_FILE=/etc/buildkit/certs/ca.crt (TS-017)
Jenkins Trivy/Harbor API 호출 → SSL_CERT_FILE 동일 경로 사용
Kubernetes Node containerd    → ca_trust Role로 OS Trust Anchor 배포 (TS-024)
```

동일한 `x509: certificate signed by unknown authority` 증상이라도 검증 주체가 다르면 별도 사건으로 관리한다.

판정: `Validated`.

---

## 4.3 Jenkins Runtime

| 항목 | 현재 값 |
| --- | --- |
| Namespace | `cicd` |
| Controller Image | `jenkins/jenkins:2.568.2-lts-jdk21@sha256:25a0…00d3`(Digest 고정) |
| Home | `jenkins-home` PVC(`nfs-k8s` StorageClass) |
| 구성 | JCasC(`jenkins-jcasc` ConfigMap), Setup Wizard 비활성 |
| Plugin | `plugins-lock.txt` 76개(전이 의존성 포함) 고정(GitOps PR #65) |
| Priority | `cicd-low-priority`(-1) — Runtime 자원 경합 시 CI가 먼저 양보 |
| Argo CD | `jenkins` Application(`cicd/`), `prune`·`selfHeal` |

### Agent PodTemplate

| Template | 컨테이너 | 용도 |
| --- | --- | --- |
| `buildkit-rootless` | `buildkit`(v0.32.2-rootless), `python`(ci-python, Trivy 포함), `node` | main Image Build·Scan·Harbor API |
| `app-ci-check` | `app-ci`(Backend/Frontend/Browser 검사 통합 Image) | PR/main P1 검사, GitOps Promotion |
| `buildkit-rootless-pr` | `buildkit` | PR Build Verify(Push 없음) |

모든 Agent Image는 Tag가 아니라 Digest로 고정한다. Rootless BuildKit은 `seccompProfile: Unconfined` + `BUILDKITD_FLAGS=--oci-worker-no-process-sandbox` 조합을 BuildKit 컨테이너에만 적용한다(TS-022).

### Credential 주입 경계

```text
Vault (seokpan-infra group_vars/all/vault.yml)
    ↓ jenkins_secrets Role (Ansible, no_log)
Kubernetes Secret (cicd ns)
    ↓ jenkins-controller-deployment.yaml secretKeyRef → env
JCasC credentials: ${ENV_VAR} 치환
    ↓
Jenkins Credential ID
    ↓
Jenkinsfile credentialsId 참조
```

| Credential ID | K8s Secret | 용도 |
| --- | --- | --- |
| `harbor-robot-account` | `jenkins-harbor-credential` | Push/Pull, Trivy |
| `harbor-api-robot-account` | `jenkins-harbor-api-credential` | Harbor API |
| `gitops-promotion-github` | `jenkins-gitops-promotion-credential` | GitOps Promotion PR |
| (파일 마운트) | `harbor-robot-dockerconfig`(Opaque, `config.json`) | BuildKit `DOCKER_CONFIG`(TS-020) |

판정: `Validated`.

---

## 4.4 CI Pipeline

### PR Pipeline (`Jenkinsfile`)

```text
App CI Check (app-ci-check)
  Checkout → Browser Cache → Backend Install & Verify → OpenAPI Export
  → Frontend Install & Verify → Browser UI → Browser Full → Aggregate
Build Verify (buildkit-rootless-pr)
  Checkout(pinned SHA) → Backend Build Verify → Frontend Build Verify
```

PR에서는 Harbor Push·GitOps 변경·실제 Provider Credential을 사용하지 않는다.

### main Image Pipeline (`Jenkinsfile.image-pipeline`)

```text
Branch Guard & Prepare (main 외 실행 금지, SHA 고정)
→ P1 Revalidation (main Commit 재검사)
→ Guard: Final Tag Check (기존 immutable Final 존재 시 재사용)
→ Build & Push Candidate (+ SBOM/Provenance, push-time Digest 캡처)
→ Scan (Trivy: CRITICAL 또는 fixable HIGH 발견 시 Promote 금지)
→ Candidate Health Smoke (/health/live HTTP 200, checker 컨테이너)
→ Promote (git-<sha12> Final Tag 부여)
→ Verify Digest & Cleanup Candidate (scan-* Tag-only 정리)
→ Record Commit-Build-Tag-Digest Evidence (image-metadata.json)
→ Create GitOps Promotion PR (4.5절)
```

Fail-closed 원칙:

- PR context / non-main 실행 금지
- P1 실패 시 Image 작업 금지
- Scan·Smoke 실패 시 Promote 금지
- Harbor API / Guard / Digest 오류 시 중단
- `disableConcurrentBuilds()` — 동일 Pipeline 동시 실행 금지

`image-metadata.json`은 `commit_sha_full`, `jenkins_build_number`, `run_id`, 컴포넌트별 `candidate_digest`·`final_digest`·`scan`·`health_smoke`·`cleanup_status`를 기록하고 `fingerprint: true`로 보관한다.

판정: `Validated`(main #18 Backend/Frontend Scan PASS·Health Smoke PASS·Promote 200·Evidence 기록 확인, 이후 최종 Source Freeze Commit까지 동일 Pipeline 사용).

---

## 4.5 GitOps Promotion 자동화

### 책임 경계 변경 이력

```text
2026-09-11 PROJECT_CHANGES
  Jenkins는 Image Tag/Digest/Evidence 생성에서 책임 종료
  GitOps PR 자동 생성은 MVP 책임 경계에서 제외
  "향후 확장 시 Credential 범위·write 권한·Review Gate·실패/재시도 경계를 다시 검토"
        ↓
2026-09-18 seokpan-app#98 / seokpan-gitops#57 Step 9
  위 재검토 조건을 충족하는 형태로 확장:
  - 별도 Bot 계정(seokpan-ci-bot) + seokpan-gitops 한정 fine-grained PAT
    (Contents/Pull requests R/W, 조직 관리자 승인)
  - PR 생성까지만 자동화, auto-merge·main 직접 push·Cluster 직접 변경 금지
  - Branch Protection(Review 1건 필수)을 그대로 Review Gate로 사용
  - 중복/충돌 PR은 fail-closed로 사람 확인 요구
```

> **주의**: 이 확장은 2026-09-11 결정을 "뒤집은" 것이 아니라 그 결정이 남긴 재검토 조건을 충족해 범위를 넓힌 것이다. 다만 `PROJECT_CHANGES.md`에 후속 항목이 아직 없으므로 10.2절 Gap으로 관리한다.

### 동작 계약 (`scripts/promote_gitops.py`)

```text
Source Checkout을 SEOKPAN_GIT_SHA로 고정 (불일치 시 중단)
→ GitOps main을 읽어 Branch Base로 사용
→ Deployment metadata annotation seokpan.io/app-source-commit에서 현재 배포 Source SHA 확인
→ 배포 SHA..현재 SHA 사이 변경 경로로 Component 영향 판정 (COMPONENT_PATHS)
→ 영향받은 Component만 kustomization.yaml digest + annotation 갱신
→ Promotion Branch Push → PR 생성
```

| 결과 코드 | 의미 |
| --- | --- |
| `PROMOTION_NO_CHANGE` | 이미지 입력 경로 변경 없음(예: CI 파일만 변경) — PR 생성 안 함 |
| `PROMOTION_PR_URL` | PR 생성 완료 |
| `PROMOTION_OPEN_PR_REQUIRES_REVIEW` | 같은 대상의 open PR 존재 — 자동 재사용하지 않고 사람 확인 요구 |
| `PROMOTION_CLOSED_UNMERGED` | 과거 PR이 merge 없이 닫힘 — 사람 확인 요구 |
| `PROMOTION_ERROR` | 그 외 오류, 중단 |

실행 Agent는 Harbor Credential이 마운트되지 않는 `app-ci-check`이며, `gitops-promotion-github`만 명시적으로 바인딩한다. 실행 전 `test_promote_gitops.py`(9건)를 Production Agent에서 먼저 통과해야 한다.

### 최종 E2E 결과

| 순서 | App Source Merge | GitOps Promotion PR Merge | 대상 |
| --- | --- | --- | --- |
| 1 | `b774a40` 2026-09-22 19:06 (App #107, Source Freeze) | GitOps #132 19:18 | backend + frontend |
| 2 | `4b803c3` 2026-09-22 20:10 (App #109) | GitOps #133 20:21 | frontend |
| 3 | `b281976` 2026-09-22 22:16 (App #111) | GitOps #134 2026-09-23 00:20 | frontend |

세 PR 모두 `seokpan-jenkins` Bot이 Co-author로 기록된 자동 생성 PR이며, Component 영향 판정대로 Backend 변경이 없는 Commit은 Frontend만 Promotion됐다. 첫 실행(App PR #103 merge, Build #20)에서는 CI 파일만 바뀌어 `PROMOTION_NO_CHANGE=1`로 올바르게 판정됐다.

판정: `Validated`.

---

## 4.6 Argo CD / GitOps CD

| 항목 | 현재 값 |
| --- | --- |
| 버전 | v3.4.7(`argocd_bootstrap` Role, Ansible 소유) |
| 구조 | Root Application → `argocd/applications/`(Child Application CR) → 각 경로 |
| 동기화 | `automated: prune: true, selfHeal: true` |
| server | `server.insecure=true`(Gateway TLS Terminate 구조 대응, Infra PR #209) |
| EndpointSlice | `argocd-cm resource.exclusions`에서 EndpointSlice 제외 해제(Infra PR #196) — 외부 Exporter EndpointSlice를 Git으로 관리하기 위한 선행조건 |
| Bootstrap 검증 | application-controller StatefulSet Ready Validation(Infra PR #193), SSA 실제 적용(`apply: true`, Infra PR #202) |

본 영역이 소유한 Child Application:

| Application | Path | 비고 |
| --- | --- | --- |
| `observability` | Helm Chart + `$values` + `observability/` Multi-source | `ServerSideApply=true` |
| `jenkins` | `cicd/` | |
| `argocd-webhook-access` | `platform/argocd-webhook-access/base` | ReferenceGrant만 소유(범위 축소 — Ansible 소유 리소스와 selfHeal 충돌 방지) |

### Self-Heal / Rollback 실측 (seokpan-gitops#57)

```text
Step 7 Self-Heal
  frontend를 kubectl scale 2→3으로 Live Drift 유발
  → Argo CD OutOfSync 감지 → 수 초 내 2로 자동 복귀
  → Git에 선언되지 않은 annotation만 추가한 경우에는 Self-Heal이 동작하지 않음
    (Argo CD는 Git에 선언된 필드만 비교함을 확인)

Step 8 Git Revert Rollback
  GitOps #119 (2026-09-18 10:04) 존재하지 않는 Digest 주입
  → apps-frontend 새 Pod ImagePullBackOff
  → GitOps #120 (10:15) git revert Merge
  → apps-frontend Synced/Healthy, Deployment 2/2, Pod Digest가 기존 Baseline과 일치
    (sha256:123203a4…0b0701)
```

Step 8에서 기본 RollingUpdate 전략이 새 ReplicaSet 검증 전에 기존 ReplicaSet을 축소해 가용 Pod가 2/2 → 1/2로 떨어지는 것을 관찰했다. 이는 Rollback 경로의 결함이 아니라 Deployment 전략 개선 후보(`maxUnavailable: 0`)이며 Backend/Frontend Desired State Owner(정태훈)에게 인계한다(10.4절).

판정: `Validated`(Sync / Self-Heal / Git Revert Rollback).

---

## 4.7 Argo CD Webhook 변경 감지 (seokpan-gitops#117)

Argo CD 기본 Polling(약 3분)으로 인한 Merge → Sync 지연을 줄이기 위해 GitHub Webhook을 연결했다.

```text
GitHub seokpan-gitops push event
    ↓ https://game.seokpan.soldesk.store/api/webhook (POST, Exact path)
seokpan-gateway (TLS Terminate)
    ↓ HTTPRoute argocd-webhook-route (application ns)
    ↓ ReferenceGrant allow-application-httproute-to-argocd-server (argocd ns)
argocd-server (insecure, HTTP)
    ↓ X-Hub-Signature-256 검증 (argocd-secret webhook.github.secret)
Application Refresh / Sync
```

- 1단계: `server.insecure=true`(Infra PR #209)
- Webhook Secret: `argocd_webhook_secret` Role — `argocd-secret` 전체가 아니라 `webhook.github.secret` 필드만 JSON merge patch(Infra PR #210). Route 공개 전에 먼저 적용해 서명 없는 Trigger를 차단한다.
- 2단계: HTTPRoute + ReferenceGrant(GitOps PR #123). 기존 `game` Route hostname을 재사용하고 path·method를 제한해 기존 라우팅 회귀가 없도록 정태훈과 합의했다.

판정: `Validated`.

---

# 5. Observability Current State

## 5.1 Stack 구성과 저장소

| 구성 | 방식 | 저장/보존 |
| --- | --- | --- |
| Prometheus | kube-prometheus-stack(Helm `88.5.4`) | `worker-01` 정적 Local PV, 7일 / 20Gi, Node 종속(유실 수용) |
| Alertmanager | kube-prometheus-stack | PVC 없음 |
| Grafana | kube-prometheus-stack, Sidecar Provisioning | PVC 없음, ConfigMap으로만 복원 |
| Loki | Raw Manifest, Single Binary `3.7.6` | `worker-02` 정적 Local PV, 72h / 10GiB |
| Alloy | Raw Manifest DaemonSet `v1.6.1` | 5 Node 전체 |

Prometheus와 Loki를 서로 다른 Worker에 고정한 이유는 Node 종속 Local PV 장애의 영향을 각각 독립적으로 관찰하기 위함이다(PROJECT_CHANGES Prometheus·Loki Local PV 배치 확정).

Argo CD `observability` Application은 Helm Chart(kube-prometheus-stack)와 Raw Manifest(`observability/`)를 하나의 Multi-source Application으로 관리한다. Helm이 관리하지 않는 Loki·Alloy·Dashboard·ServiceMonitor·NetworkPolicy·Alertmanager 설정 Secret은 `observability/` 경로에 둔다.

판정: `Running / Validated`.

---

## 5.2 Metric 수집

### In-cluster

초기에는 Prometheus Target 19건이 `context deadline exceeded`로 Down이었다. 원인은 세 갈래였고 각각 해결했다.

```text
(1) Prometheus → kube-apiserver(6443) Egress 누락 → activeTargets=0 (Service Discovery 불가)
(2) CoreDNS(9153)/Grafana(3000)/operator(10250) Egress, Alertmanager config-reloader(8080) Ingress 누락
(3) controller-manager/scheduler/etcd/kube-proxy 직접 Scrape
    → G-07 요구 대상 아님 + kubeadm bind-address 재구성 비용 과다
    → Issue #177 결정으로 Chart에서 비활성화, 관련 Egress도 최소권한 원칙으로 제거
```

- ServiceMonitor 선택 label: `release: kube-prometheus-stack`
- Application Metrics: Backend `/metrics` ServiceMonitor 활성화, 2 Replica Target UP, 실제 Metric Query 확인(GitOps #91 / PR #107, 정태훈과 Cross-role)
- Alloy 자체 Metric: `servicemonitor-alloy.yaml`

### 외부 VM Exporter

외부 VM은 Kubernetes Service + 수동 `EndpointSlice` + ServiceMonitor 조합으로 수집한다.

| Job | 대상 | 결과 |
| --- | --- | --- |
| `node-exporter-external` | VRouter-01~04(`192.168.51~54.10`), MaxScale-01, NFS, Ansible Controller | **7/7 UP** |
| `mariadb-exporter` | mariadb-01(`52.40`), mariadb-02(`51.40`) | **2/2 UP**(Exporter 설치·계정은 김상희) |
| `harbor-metrics` | `harbor-01:9090` | **1/1 UP** |
| `lb01-haproxy-exporter` | lb01 `10.1.93.78:8404` | HAProxy 2.0+ 네이티브 Prometheus 모듈(Infra PR #208, GitOps #111/#112) |

외부 Target 정상화 과정의 주요 판단:

- **Target 0/0의 실제 원인은 EndpointSlice 부재가 아니라 Exporter 미설치**였다(seokpan-gitops#103에서 최초 전제 정정). 김상희가 작성한 `node_exporter_linux` Role을 수정 없이 재사용해 VRouter 4대 + Ansible Controller에 설치했다.
- vrouter-02/03/04가 `no route to host`로 Down이던 원인은 VRouter ACL이 아니라 **9100/tcp가 internal zone에만 열리고 실제 유입 인터페이스인 external zone에는 규칙이 없던 것**이었다(Issue #205, 이유빈 진단). 전체 개방 대신 Worker 대역(`192.168.51.0/24`, `192.168.52.0/24`)만 rich rule로 허용했고, 리뷰에서 "단독 fix Playbook은 VRouter 재구축 시 재발"이라는 지적을 받아 기존 `vrouter_firewall` Role에 통합했다(Infra PR #206).
- lb01은 `haproxy.cfg`에 stats 섹션이 없었다. 2023년 이후 갱신이 멈춘 별도 `haproxy_exporter` 바이너리 대신 HAProxy 네이티브 `prometheus-exporter` 서비스를 사용하고, 8404/tcp를 VIP가 아닌 실IP(`10.1.93.78`)에만 바인딩해 기존 4개 서비스 Listener와 분리했다.

판정: `Validated`.

---

## 5.3 Log 수집

```text
Pod stdout/stderr
    ↓ loki.source.file "pod_logs" (Alloy DaemonSet, /var/log/pods)
    ↓ labels: namespace / pod / container / node_name
Node systemd journal
    ↓ loki.source.journal "node_journal"
    ↓ labels: node_name / unit / level
Loki (Single Binary)
    ↓
Grafana Loki Datasource
```

- Alloy DaemonSet DESIRED/READY 5/5, `totalLinesProcessed` 증가 확인(GitOps PR #81)
- Node Journal 5 Node(cp-01/02/03, worker-01/02) 연속 수집, 기존 Pod Log 경로 회귀 없음(GitOps PR #99)
- Loki Query API(`query_range`, `labels`) HTTP 200 / `status: success`
- Loki PVC 참조 불일치(TS-029), compactor `delete_request_store` 누락(TS-030)은 해결됨

노드가 SELinux enforcing이라 hostPath 접근은 `runAsUser: 0`만으로 충분하지 않다. Journal 수집 추가 전 격리 Canary Pod로 AVC Denial 여부를 먼저 확인한 뒤 DaemonSet에 반영했다.

**외부 VM Log**: 06 문서가 정의한 `alloy_linux`(외부 CentOS Guest → Loki NodePort 31000) 경로는 현재 `seokpan-infra`에 Role이 없고 Loki 외부 노출도 구성되지 않았다. 외부 VM은 Metric만 수집하며 Log는 수집하지 않는다(10.3절).

판정: `Validated`(In-cluster Pod Log·Node Journal) / `Not Implemented`(외부 VM Log).

---

## 5.4 Alert / E-mail

### SMTP Provider 결정 이력

```text
SendGrid / Resend(도메인 없음 전제) 기각
→ Mailgun Sandbox 검토
→ Gabia에서 soldesk.store 소유 확인
→ Resend 확정 (도메인 인증 가능)
```

### 구성

| 항목 | 값 | 소유 |
| --- | --- | --- |
| Smarthost | `smtp.resend.com:587`(STARTTLS, `smtp_require_tls: true`) | GitOps |
| From | `alertmanager@alerts-seokpan.soldesk.store` | GitOps |
| Username | `resend`(고정 문자열) | GitOps |
| Password | Resend API Key → `alertmanager-smtp-credential` Secret(`smtp-password`) | Ansible(`alertmanager_secret`, Vault `vault_alertmanager_resend_api_key`) |
| 수신자 | `seokpan@soldesk.store`(팀 Zoho Mail) | GitOps |
| DNS | SPF / DKIM(Gabia, `alerts-seokpan.soldesk.store` 도메인 인증) | 본 영역(DNS 등록) |

`alertmanager-config` Secret(`alertmanager.yaml` 소유)과 Credential Secret은 분리했다. Credential은 `alertmanagerSpec.secrets`로 `/etc/alertmanager/secrets/alertmanager-smtp-credential/`에 마운트하고, 설정은 `smtp_auth_password_file`로 경로만 참조한다. `configSecret: alertmanager-config`를 명시해 Chart 기본 이름과의 불일치를 막았다.

### Routing

```text
default          → default-email   (group_wait 30s, group_interval 5m, repeat 4h)
Watchdog / InfoInhibitor → null    (생존 신호용 메타 Alert, 메일 미발송 — GitOps PR #129)
severity=critical → critical-email (repeat 30m)
send_resolved: true (양쪽 Receiver)
```

### 검증 결과

- `vector(1)` 테스트 PrometheusRule로 firing → resolved API 사이클 확인
- Alertmanager Pod의 `smtp-password` 마운트, `/api/v2/status` 설정 확인
- Test Alert firing / resolved **E-mail 실제 수신**, DKIM/SPF/DMARC 모두 pass
- 첫 메일은 스팸함 도착 — 신규 도메인 평판(warm-up) 문제로 판단, 인증 실패 아님
- `alertmanager_smtp_preflight` Role로 DNS/TCP/STARTTLS를 사전 검증하고 block/always로 Cleanup 보장(Infra PR #189)

**Alert Rule 범위**: 현재 평가되는 Rule은 kube-prometheus-stack `defaultRules`(etcd·controller-manager·scheduler·kube-proxy 제외)다. 프로젝트 전용 PrometheusRule(MariaDB 복제 중단·지연, M-축 KPI 등)은 작성되지 않았고, `critical-email` Route는 해당 Rule이 추가되면 바로 쓸 수 있는 뼈대 상태다(10.3절).

판정: `Validated`(E-mail 통보 경로) / `Partial`(프로젝트 전용 Alert Rule).

---

## 5.5 Grafana Dashboard

모든 Dashboard와 Datasource는 `grafana_dashboard: "1"` / `grafana_datasource: "1"` label ConfigMap으로만 Provisioning하며, UI 직접 변경은 결과로 인정하지 않는다.

| Dashboard | 관점 | 주요 패널 | 근거 |
| --- | --- | --- | --- |
| Kubernetes / Infrastructure Overview | Cluster | kube-apiserver Request/5xx/p99, Node CPU/Memory/Disk, Pod Restart/Phase, Node Ready | GitOps PR #73 |
| External VM Overview | 외부 인프라 | Target Status, CPU, Load, Memory, Disk IO/Space, Network | GitOps PR #114 |
| MariaDB Overview | Data | Target Status, Connections, QPS, Slow Queries, Buffer Pool, Uptime, Aborted Connections | GitOps PR #114 |
| Service / KPI Overview | 서비스 | Vote 처리량·5xx 오류율·p95/p99, M-01 충돌/Stale 거부(409), Backend route별 분포 | GitOps PR #114 |

설계 원칙:

- 외부 VM·MariaDB 패널은 `and on(instance) up{job=...} == 1` 조인으로 Down 인스턴스를 자동 제외한다. kube-prometheus-stack 기본 mixin Dashboard는 `job="node-exporter"`로 고정되어 외부 VM(`node-exporter-external`)이 보이지 않기 때문에 전용 Dashboard를 만들었다.
- Service / KPI Overview는 실제 계측된 Metric(`seokpan_http_requests_total`, `seokpan_http_request_duration_seconds`)만 사용한다. 존재하지 않는 Metric 이름은 전량 제거했고, 12 문서 3.2절(임의 목표값 금지)에 따라 색상 임계값을 두지 않는다. VoteRuleViolation 계열이 모두 HTTP 409로 매핑되는 점을 이용해 M-01 "충돌 거부" 축을 실시간 대리 신호로 표시한다(M-01 전체 판정은 대체하지 않음).
- Datasource `isDefault: true` 중복 충돌(GitOps PR #77), Grafana sidecar의 kube-apiserver Egress 누락(GitOps PR #75)을 해결해 Provisioning이 실제 반영됨을 확인했다.

멘토링 권고(서비스 관점과 클러스터 관점 Dashboard 분리)는 충족됐다.

판정: `Validated`.

---

## 5.6 Observability NetworkPolicy

`observability/networkpolicy-observability.yaml`은 번호가 붙은 정책마다 Egress/Ingress를 대칭 쌍으로 정의한다.

```text
1) Prometheus → in-cluster (kube-state-metrics, Alertmanager, CoreDNS, Grafana, operator, App 8000)
2) Prometheus → kubelet 10250 (CP/Worker 5 Node /32)
3) Prometheus → 외부 Exporter (9100 / 9104 / 8404 / 9090)
4) Grafana → Prometheus 9090 / Loki 3100 / kube-apiserver 6443
5) Alloy → Loki 3100
6) Alertmanager → SMTP 587/465 (0.0.0.0/0 — Resend 고정 IP 없음)
7~9) 각 Egress에 대응하는 Ingress
+ Prometheus → kube-apiserver 6443 (CP 3대 /32)
```

DNS Egress는 `kube-system` namespace로 좁혔다. 최초 파일은 Egress만 있고 Ingress가 없어 Default Deny 전환 시 내부 통신이 전부 막히는 상태였으며, 2026-09-03에 Ingress 쌍을 추가했다.

판정: `Validated`(observability namespace 범위).

---

# 6. Provider / Consumer Contract

## 6.1 본 영역이 제공하는 것

| Provider | 제공 내용 | Consumer |
| --- | --- | --- |
| Harbor | Private Registry, 검증 Final Digest | Jenkins, Kubernetes Node(containerd) |
| Harbor Robot(CI/API) | Push/Pull·API Credential | Jenkins Pipeline |
| Jenkins Image Pipeline | `image-metadata.json`(Commit→Build→Tag→Digest), SBOM/Provenance, Scan 결과 | GitOps Promotion, 발표·Evidence |
| GitOps Promotion | Digest + `seokpan.io/app-source-commit` 갱신 PR | GitOps Reviewer(정태훈/팀) |
| Argo CD | Desired State Sync, Self-Heal | 모든 Child Application Owner |
| Prometheus | Metric 저장·Query | Grafana, 12 문서 측정, 팀 전체 |
| Loki | Log 저장·Query | Grafana, 장애 Timeline |
| Alertmanager | E-mail 통보 | 팀 전체(`seokpan@soldesk.store`) |
| Grafana | Dashboard 4종 | 발표·검증·장애 분석 |

## 6.2 본 영역이 소비하는 것

| 소비 대상 | Provider | 계약 |
| --- | --- | --- |
| Backend `/metrics` | 정태훈(`seokpan-app#78`) | Service label `app.kubernetes.io/name: backend`, port `http:8000`, path `/metrics` |
| Application stdout Log | 정태훈(`seokpan-app#92`) | 민감정보 Raw Log 미출력 |
| mysqld_exporter / node_exporter(DB·NFS) | 김상희(Infra #199/#201) | 9104 / 9100, `exporter_svc` 계정(`SLAVE MONITOR` 포함, TS-043) |
| VRouter·LB 방화벽 / HAProxy 설정 | 이유빈 | Worker 대역 → 9100·8404 허용 |
| Kubernetes Node·Gateway | 정태훈 | Webhook HTTPRoute가 `seokpan-gateway` 재사용 |
| Ansible 실행환경 | 이유빈 | ansible-core 2.20.8 / kubernetes.core 6.5.0 / kubernetes client 36.0.3 / Python 3.12.13, `ansible-safe-run` |
| Cluster kubeconfig | 이유빈 | `cluster_kubeconfig: /etc/seokpan/kubeconfig/admin.conf` |

```text
Provider 완료 ≠ "내 쪽에서 동작"
Provider 완료 = Consumer가 실제로 사용해 결과를 확인한 상태
```

예: Prometheus가 Running이어도 Backend Target UP + 실제 Query 결과가 확인되기 전에는 Application Metrics 수집 완료로 보지 않았다.

---

# 7. Integration 흐름

## 7.1 Delivery E2E (최종 확인 경로)

```text
seokpan-app main Merge (예: b774a40, App #107)
→ Jenkins main Image Pipeline
   (P1 → Build/SBOM → Candidate Push → Trivy → Smoke → Promote → Digest Verify → Evidence)
→ Create GitOps Promotion PR (seokpan-ci-bot, app-ci-check Agent)
→ seokpan-gitops PR (#132) Review / Squash Merge
→ GitHub Webhook → Argo CD Refresh / Sync
→ apps-backend / apps-frontend Synced / Healthy
→ Pod ImageID = Harbor Final Digest
```

## 7.2 Metric / Alert

```text
Target (in-cluster / 외부 VM / Harbor / LB)
→ Prometheus Scrape (ServiceMonitor, release: kube-prometheus-stack)
→ Rule 평가 → Alertmanager (NetworkPolicy 9093)
→ Resend SMTP 587 → seokpan@soldesk.store
→ firing / resolved
```

## 7.3 Log

```text
Pod / Node Journal → Alloy DaemonSet → Loki 3100 → Grafana Explore
```

## 7.4 Rollback

```text
문제 GitOps Revision
→ git revert PR → Merge
→ Argo CD Sync
→ 기존 Digest 복귀 확인
(kubectl edit / rollout undo 사용 안 함 — selfHeal이 즉시 되돌림)
```

---

# 8. Integration Gate

## Gate A — Registry Ready

### 확인 내용

* Harbor 설치·재부팅 후 자동 기동
* Private Project, TLS(SAN), 내부 CA 신뢰(BuildKit·containerd)
* Robot Account 권한 분리(CI / API / Runtime Pull)
* Tag Immutability + `scan-*` 예외
* GC Policy

### 판정

```text
Harbor + TLS/CA + Robot 분리 + Immutability
→ PASS
```

---

## Gate B — CI Build & Artifact Ready

### 확인 내용

* JCasC·Plugin Lock·Agent Image Digest 고정
* Credential 3단 주입 경로
* PR Pipeline(검사·Build Verify, Push 없음)
* main Pipeline(Build/SBOM/Scan/Smoke/Promote/Digest Verify/Evidence)
* Fail-closed 동작

### 판정

```text
Commit → Jenkins Run → Harbor Final Digest → image-metadata.json
→ PASS
```

---

## Gate C — GitOps CD Ready

### 확인 내용

* Argo CD Bootstrap(Ansible) 재실행 멱등성
* App-of-Apps 편입
* Digest Pinning 배포 → Pod ImageID 일치(Cross-role, KAI-DEL-01)
* Self-Heal(Step 7)
* Git Revert Rollback(Step 8)

### 판정

```text
GitOps Revision → Argo CD Sync → Pod Ready
Live Drift → Self-Heal
Bad Revision → git revert → Baseline Digest 복귀
→ PASS
```

---

## Gate D — Delivery Automation (G-06 E2E)

### 확인 내용

* Jenkins → GitOps Promotion PR 자동 생성(PR #132~#134)
* Component 영향 판정 / NO_CHANGE 판정
* 중복·충돌 PR fail-closed
* auto-merge 없음, Review Gate 유지
* Webhook 기반 변경 감지

### 판정

```text
App Commit → Digest → GitOps PR → Merge → Webhook → Sync → Healthy
→ PASS
```

---

## Gate E — Metric Collection

### 확인 내용

* In-cluster Target 정상화(19건 Down 해소)
* Application Metrics(Cross-role)
* 외부 VM node_exporter 7/7, mariadb-exporter 2/2, Harbor 1/1, lb01 HAProxy

### 판정

```text
Target UP + 실제 Query 결과
→ PASS
```

---

## Gate F — Log Collection

### 확인 내용

* Alloy DaemonSet 5/5
* Pod Log(namespace/pod/container/node_name label)
* Node Journal(node_name/unit/level label)
* Loki Query API 200

### 판정

```text
In-cluster Pod Log + Node Journal
→ PASS

외부 VM Log (alloy_linux)
→ Not Implemented
```

---

## Gate G — Alert / Dashboard (G-07)

### 확인 내용

* Alertmanager firing → resolved API
* E-mail 실제 수신(firing/resolved), DKIM/SPF/DMARC pass
* Watchdog 메일 억제
* Dashboard 4종 Provisioning, Cluster/서비스 관점 분리

### 판정

```text
Prometheus → Alertmanager → E-mail (firing/resolved)
Grafana Provisioning Dashboard
→ PASS

프로젝트 전용 Alert Rule (복제 지연, M-축)
→ Partial / Deferred
```

---

# 9. Gate 요약

| Gate | 대상 | 현재 상태 | 핵심 Evidence |
| --- | --- | --- | --- |
| A | Harbor Registry | `Validated` | TS-010/017/024/027/034, Infra PR #21/#69/#157/#161 |
| B | CI Build & Artifact | `Validated` | App PR #68, main Build #18, TS-020/022/028/035 |
| C | GitOps CD | `Validated` | seokpan-gitops#57 Step 2~8, GitOps PR #119/#120 |
| D | Delivery Automation(G-06) | `Validated` | App PR #103, GitOps PR #127, #132~#134, GitOps #117/#123 |
| E | Metric | `Validated` | GitOps PR #70/#71/#106/#107, #103/#205, Infra PR #206/#208 |
| F | Log(In-cluster) | `Validated` | GitOps PR #30/#81/#99 |
| F | Log(외부 VM) | `Not Implemented` | `alloy_linux` Role 부재 |
| G | Alert E-mail / Dashboard(G-07) | `Validated` | GitOps PR #97/#101/#129/#73/#114, Infra PR #187/#189 |
| G | 프로젝트 전용 Alert Rule | `Partial` | defaultRules만 평가, 전용 Rule 2차 |

```text
M4 Delivery/Observability (07 문서)
→ Harbor · Jenkins · Argo CD · Metric · Log · E-mail 연결
→ 충족
```

---

# 10. 남은 Blocker와 Gap

MVP Acceptance를 막는 **Blocker는 없다**. 아래는 "완료"로 확대 해석하지 않기 위해 분리해 두는 Gap이다.

## 10.1 장애 주입 시험 미실행 (06 F-10~F-13)

| ID | 시나리오 | 상태 |
| --- | --- | --- |
| F-10 | Harbor 중단 — 실행 Pod 유지 vs 신규 Pull/Deploy 실패 분리 | `Not Tested` |
| F-11 | Jenkins Controller/Agent 중단 — Runtime 무영향·CI 재개·PVC 재연결 | `Not Tested`(TS-035에서 우발적 Controller 다운·복구 1회 관찰, 계획 시험 아님) |
| F-12 | Alertmanager 중단 — 게임 유지·통보 공백·복구 후 resolved | `Not Tested` |
| F-13 | Prometheus/Loki Node 유실 — 관측 공백·Local 데이터 유실 | `Not Tested` |

## 10.2 문서 기준 현행화

```text
PROJECT_CHANGES 2026-09-11 "Jenkins는 GitOps PR 생성 안 함"
→ 2026-09-18 App #98로 조건부 확장(4.5절)
→ PROJECT_CHANGES 후속 항목 미작성
```

`seokpan-gitops` Manifest 주석 중 이미 해소된 TODO가 남아 있다(예: `alertmanager-config.yaml` 상단 "SMTP 서버 미확정", `jenkins.yaml` "갭 5건 Issue 번호 기입", NetworkPolicy의 `8000 [TODO]`·`loadgen` 항목). 동작에는 영향이 없으나 현행 상태와 다르므로 정리 대상이다.

## 10.3 Observability 잔여 범위

| 항목 | 상태 | 비고 |
| --- | --- | --- |
| 외부 VM Log 수집(`alloy_linux`) | `Not Implemented` | Role·Loki 외부 노출 모두 없음 |
| 프로젝트 전용 PrometheusRule | `Partial` | 복제 중단·지연 Rule은 PROJECT_CHANGES 2026-09-21에서 2차 이관 확정 |
| MariaDB Replication 패널 | `Deferred` | TS-043 수정으로 복제 Metric 조회 가능해졌으므로 2차에서 패널 추가 가능 |
| maxscale_exporter | `Deferred` | Data 영역 소유, 소스 빌드 방식 2차(TS-045) |
| Grafana / Jenkins 외부 Route | `Not Implemented` | `grafana.seokpan.soldesk.store`는 이름 해석만 적용, Jenkins는 미적용 — 현재 port-forward로 접근 |

## 10.4 Delivery 잔여 범위

| 항목 | 상태 | Owner |
| --- | --- | --- |
| Harbor Retention 정책(실패 Build의 `scan-*` Candidate 정리) | `Not Implemented` | 본 영역 |
| Jenkins Job/Trigger 정의의 Git 선언 | `Not Implemented` — JCasC에 Job 선언 없음 | 본 영역 |
| 레거시 `jenkins-github-gitops-credential` Secret 정리 | `Not Done` — canonical `jenkins-gitops-promotion-credential` 전환 후에도 `jenkins_secrets` Role에 유지 중(Consumer 없음) | 본 영역 |
| Deployment `maxUnavailable: 0`(Step 8 관찰) | 후보 인계 | 정태훈 |
| 06 "60 Delivery / 70 Platform" 통합 Playbook | `Deferred` — 현재 Role별 개별 Playbook | 이유빈·본 영역 |
| Q-07 CI/CD 수동 대비 시간 비교 | `Not Tested`(PR Merge 시각만 존재, 12 문서 DOB-Q07-01) | 본 영역 |

## 10.5 보안 Gap (공유 사항)

`application`, `argocd`, `cicd` namespace에는 현재 NetworkPolicy가 없고 Calico GlobalNetworkPolicy도 없어 Default Deny가 실제로는 적용되지 않는다. Observability namespace 정책은 Default Deny 전환을 전제로 작성되어 있으나, 클러스터 전체 Default Deny 적용 여부는 Network 담당 결정 사항이다.

---

# 11. Critical Path

```text
Internal CA / TLS
    ↓
Harbor (Robot · Immutability)
    ↓
Jenkins (JCasC · Credential · BuildKit)
    ↓
main Image Pipeline (Scan · Smoke · Digest · Evidence)
    ↓
Argo CD Bootstrap · App-of-Apps
    ↓
Digest Pinning 배포 · Self-Heal · Rollback (완료)
    ↓
GitOps Promotion 자동화 · Webhook (완료)
    ↓
Prometheus / Loki / Alloy / Alertmanager (완료)
    ↓
외부 VM Exporter · Dashboard (완료)
    ↓
E-mail firing / resolved (완료)
    ↓
M4 Delivery / Observability Acceptance
```

1차 프로젝트 범위에서 이 경로는 모두 실제 실행으로 닫혔다. 남은 것은 10절의 장애 주입 시험, 전용 Alert Rule, 외부 VM Log, 운영 편의 항목이며 모두 2차 프로젝트 이관 또는 후속 과제다.

---

# 12. 완료 기준

## 12.1 Delivery

* [x] Harbor 설치·자동 기동·GC
* [x] 내부 CA 신뢰(BuildKit / containerd)
* [x] Robot Account 권한 분리
* [x] Tag Immutability + `scan-*` 예외
* [x] Jenkins JCasC·Plugin Lock·Agent Digest 고정
* [x] Credential 3단 주입 경로
* [x] PR Pipeline
* [x] main Image Pipeline(Build/SBOM/Scan/Smoke/Promote/Evidence)
* [x] Argo CD Bootstrap 멱등성
* [x] Digest Pinning 배포(Cross-role)
* [x] Self-Heal
* [x] Git Revert Rollback
* [x] GitOps Promotion PR 자동화(E2E 3회)
* [x] Argo CD Webhook
* [ ] Harbor Retention
* [ ] Jenkins Job 정의 Git 선언
* [ ] F-10 / F-11 장애 시험

## 12.2 Observability

* [x] kube-prometheus-stack / Loki / Alloy Argo CD 관리
* [x] Prometheus·Loki Local PV 분리 배치
* [x] In-cluster Target 정상화
* [x] Application Metrics(Cross-role)
* [x] 외부 VM node_exporter 7/7 · mariadb 2/2 · Harbor · lb01
* [x] Pod Log / Node Journal
* [x] Alertmanager firing → resolved
* [x] E-mail 실제 수신(DKIM/SPF/DMARC pass)
* [x] Dashboard 4종(Cluster / VM / MariaDB / 서비스 KPI)
* [x] Observability NetworkPolicy
* [ ] 외부 VM Log
* [ ] 프로젝트 전용 Alert Rule
* [ ] F-12 / F-13 장애 시험

---

# 13. Traceability

## 13.1 Delivery

| 항목 | 추적 대상 | 목적 |
| --- | --- | --- |
| Harbor Role | Infra PR #21 / #47 / #69 / #74 | 설치·Robot·GC·자동 기동 |
| Harbor TLS/CA | Infra PR #34 / #87 / #127 / #140 / #163, TS-002/017/024 | 내부 CA·SAN·신뢰 경로 |
| Harbor Immutability | Infra PR #157, TS-027 / TS-034 | Final 보호, Candidate 예외 |
| Harbor API Robot | Infra PR #161, GitOps PR #49 | Pipeline API 전용 Credential |
| Jenkins Secret | Infra PR #98 / #154 / #212 / #213 | Credential 주입 |
| Jenkins Manifest | GitOps PR #19 / #34 / #37 / #41 / #54 / #62·#64·#65 | BuildKit·JCasC·Plugin Lock |
| CI Image | GitOps PR #53 / #55 / #56 | app-ci / ci-python(Trivy) |
| Pipeline | App PR #48 / #54 / #65 / #68 | Dockerfile·PR·main Pipeline |
| GitOps Promotion | App #98 / PR #103, GitOps PR #127, #132~#134 | 자동 PR·E2E |
| Argo CD | Infra PR #47 / #64 / #106 / #193 / #196 / #202 / #209 | Bootstrap·SSA·EndpointSlice |
| CD 검증 | seokpan-gitops#57, GitOps PR #119 / #120 | Self-Heal·Rollback |
| Webhook | seokpan-gitops#117, Infra PR #210, GitOps PR #123 | 변경 감지 |

## 13.2 Observability

| 항목 | 추적 대상 | 목적 |
| --- | --- | --- |
| Stack 정상화 | GitOps PR #30 / #32, TS-029 / TS-030 | Loki·Prometheus PV·Config |
| Target 정상화 | Issue #69 → GitOps PR #68 / #70, Issue #177 → PR #71 | 19건 Down 해소 |
| Grafana | GitOps PR #73 / #75 / #77 / #114 | Dashboard·Datasource |
| Log | GitOps PR #81 / #99 | Pod Log·Journal |
| Alertmanager | Infra PR #187 / #189, GitOps PR #97 / #101 / #129 | SMTP·수신자·Watchdog |
| 외부 Exporter | seokpan-gitops#103, Infra PR #204 / #206 / #208, GitOps PR #106 / #111 / #112 | VRouter·Ansible·Harbor·lb01 |
| Application Metrics | seokpan-gitops#91, GitOps PR #107 | Cross-role |

## 13.3 최종 판정

본 문서의 판단 기준은 **구현 여부가 아니라 실제 실행 결과**이며, Delivery·Observability 각각에서 "Stack이 떠 있음"과 "Consumer가 결과를 확인함"을 분리해 판정했다.

```text
M4 Delivery / Observability
= Harbor + Jenkins + Argo CD      (Validated)
+ Commit → Digest → Healthy       (Validated, G-06)
+ Metric · Log 조회               (Validated, 외부 VM Log 제외)
+ Alert firing / resolved E-mail  (Validated, G-07)
```

F-10~F-13 장애 시험, 프로젝트 전용 Alert Rule, 외부 VM Log는 실행하지 않은 항목이므로 완료로 기록하지 않으며, 12 문서에서 `Not Tested / Deferred`로 관리한다.
