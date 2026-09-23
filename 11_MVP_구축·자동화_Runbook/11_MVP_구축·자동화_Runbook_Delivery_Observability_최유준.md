# MVP 구축·자동화 Runbook
## Delivery / Observability

## 1. 목적과 사용 범위

이 문서는 「石나가는 판단」 1차 프로젝트의 Delivery / Observability 영역(Harbor, Jenkins·Rootless BuildKit, CI Pipeline, GitOps Promotion, Argo CD·Webhook, kube-prometheus-stack, Loki·Alloy, Alertmanager·E-mail, 외부 VM Exporter, Grafana Dashboard)을 실제로 구축·적용·확인·재실행·복구·Rollback하기 위한 실행 Runbook이다.

09 문서가 책임·Provider/Consumer·Integration Gate(A~G)를 정의한다면, 본 문서는 각 Gate를 실제 작업으로 통과하는 절차를 소유한다.

```text
09 = What / Why / Responsibility / Gate
11 = Pre-check / Apply / Verify / Re-run / Recovery / Rollback
12 = Test Case / Measurement / PASS·FAIL / Evidence
```

본 문서는 1차 프로젝트 종료 시점(2026-09-23)의 구현·실측 상태를 기준으로 작성한다. Delivery E2E(App Commit → Digest → GitOps PR → Argo CD → Pod)와 G-07(Metric·Log 조회, Alert firing/resolved E-mail)은 실측 완료 항목이며, 외부 VM Log 수집·프로젝트 전용 Alert Rule·Harbor Retention·장애 주입 시험(F-10~F-13)은 완료되지 않은 항목으로 별도 표시한다.

직접 기준:

- [`09_MVP_실행·통합_실시설계_Delivery_Observability_최유준.md`](../09_MVP_실행·통합_실시설계/09_MVP_실행·통합_실시설계_Delivery_Observability_최유준.md)
- [`10_GitHub_협업_및_Repository_운영.md`](../10_GitHub_협업_및_Repository_운영/10_GitHub_협업_및_Repository_운영.md)
- [`PROJECT_CHANGES.md`](../PROJECT_CHANGES.md) / [`MVP_IMPLEMENTATION_BASELINE.md`](../MVP_IMPLEMENTATION_BASELINE.md)
- `seokpan-infra` / `seokpan-gitops` / `seokpan-app` 현재 `main`
- seokpan-gitops Issue #57/#69/#91/#103/#117/#177, seokpan-infra Issue #205, seokpan-app Issue #58/#98
- TS-010/017/020/022/024/027/028/029/030/034/035

현재 상태는 계속 변할 수 있으므로 실행 직전에는 반드시 해당 Repository의 최신 `main`과 열린 Issue/PR을 다시 확인한다.

---

## 2. 실행 원칙

### 2.1 Source of Truth와 Working Directory

```text
Harbor / Argo CD Bootstrap / Secret 주입 / 외부 VM Exporter·방화벽
    → seokpan-infra (ansible/roles, ansible/playbooks, ansible/inventory)
Vault Credential
    → seokpan-infra (ansible/inventory/group_vars/all/vault.yml, ansible-vault AES256)
Jenkins / Observability / Argo CD Child Application / Webhook Route
    → seokpan-gitops (cicd/, observability/, argocd/applications/, platform/)
Pipeline 정의 / Promotion Script
    → seokpan-app (Jenkinsfile, Jenkinsfile.image-pipeline, scripts/promote_gitops.py)
Runbook / Evidence
    → seokpan-docs
```

Ansible은 Ansible Controller(`192.168.54.70`)에서 `seokpan-infra/ansible` 디렉터리 기준 `./ansible-safe-run`으로 실행한다. `ansible-safe-run`은 Vault/Become Credential을 Kernel Keyring에서 공급하므로 `-e`로 민감값을 넘기지 않는다.

### 2.2 GitOps 우선 (selfHeal)

`observability`, `jenkins`, `argocd-webhook-access`, `apps-*` Application은 모두 `selfHeal: true`다.

```text
kubectl edit / patch / rollout undo → 수 초 내 Git 상태로 되돌려짐 → 결과로 인정하지 않음
Git Branch → PR → Review → Squash Merge → Argo CD Sync → 확인  ← 유일한 영구 경로
```

진단 목적의 일시적 Live 변경(Self-Heal 시험 등)은 12 문서의 Test Case로만 수행한다.

### 2.3 상태 판정

```text
Implemented ≠ Merged ≠ Running ≠ Validated
Image Pushed ≠ Final Digest ≠ GitOps Merged ≠ Argo Synced ≠ Pod Ready(해당 Digest)
Target UP ≠ Query 결과 존재 ≠ Dashboard 표시
Alert firing ≠ E-mail 수신 ≠ resolved 수신
JCasC ConfigMap Git 반영 ≠ Jenkins Controller 반영
```

### 2.4 Secret 비노출

Robot Password, GitHub PAT, Resend API Key, Webhook Secret, Vault 평문은 콘솔·Issue·PR·문서에 출력하지 않는다. Secret은 존재 여부와 Key 이름만 확인한다. Secret을 다루는 Ansible Task는 `no_log: true`를 유지한다.

### 2.5 중단 우선

아래 조건에서는 다음 단계로 진행하지 않는다.

- 다른 사람의 open GitOps Promotion PR이 존재(`PROMOTION_OPEN_PR_REQUIRES_REVIEW`)
- Argo CD Application이 이미 `OutOfSync/Degraded`인데 원인 미확인
- Harbor API가 4xx/5xx를 반환하는데 Guard 단계를 우회하려는 시도
- Trivy Scan FAIL 상태에서 수동 Promote 시도
- NetworkPolicy 변경이 다른 namespace(application·argocd·cicd)에 영향을 주는데 소유자 미확인
- VRouter / LB / DB 서버 설정 변경을 소유자 확인 없이 진행하려는 경우(단독 결정 지양)

### 2.6 Branch 위생

항상 `main`에서 새 Branch를 만들고 즉시 다음으로 오염 여부를 확인한다.

```bash
git fetch --prune
git switch -c <type>/<issue>_<topic> origin/main
git log --oneline origin/main..HEAD   # 비어 있어야 함
```

---

## 3. 현재 실행 기준 상태

| 영역 | 상태 | 근거 |
| --- | --- | --- |
| Harbor v2.15.2 설치·systemd 자동 기동·GC | Validated | Infra PR #21/#69/#74, TS-010 |
| Harbor TLS / 내부 CA 신뢰(BuildKit·containerd) | Validated | TS-017 / TS-024 |
| Robot Account 분리(CI / API / Runtime Pull) | Validated | Infra PR #47/#161/#183 |
| Tag Immutability + `scan-*` 예외 | Validated | Infra PR #157, TS-034 |
| Jenkins JCasC·Plugin Lock(76)·Agent Digest 고정 | Validated | GitOps PR #65, TS-035 |
| Credential 주입(Vault → Secret → env → JCasC) | Validated | Infra PR #98/#154/#213, GitOps PR #127 |
| PR Pipeline / main Image Pipeline | Validated | App PR #65/#68, main Build #18 |
| Argo CD v3.4.7 Bootstrap | Validated | Infra PR #193/#196/#202/#209 |
| Digest Pinning 배포 / Self-Heal / Git Revert Rollback | Validated | seokpan-gitops#57, GitOps PR #119/#120 |
| GitOps Promotion PR 자동화 | Validated(E2E 3회) | App PR #103, GitOps PR #132~#134 |
| Argo CD Webhook | Validated | Infra PR #210, GitOps PR #123 |
| kube-prometheus-stack / Loki / Alloy | Validated | GitOps PR #30/#32/#81 |
| In-cluster Target / Application Metrics | Validated | GitOps PR #70/#71/#107 |
| 외부 VM Exporter(7/7, 2/2, Harbor, lb01) | Validated | Infra PR #204/#206/#208, GitOps PR #106/#111/#112 |
| Pod Log / Node Journal | Validated | GitOps PR #81/#99 |
| Alertmanager E-mail(firing/resolved) | Validated | Infra PR #187/#189, GitOps PR #97/#101/#129 |
| Grafana Dashboard 4종 | Validated | GitOps PR #73/#77/#114 |
| 외부 VM Log(`alloy_linux`) | Not Implemented | Role 부재 |
| 프로젝트 전용 PrometheusRule | Partial | defaultRules만 평가 |
| Harbor Retention | Not Implemented | Pipeline 주석 "Retention 미구성" |
| Jenkins Job/Trigger Git 선언 | Not Implemented | JCasC에 Job 선언 없음 |
| 레거시 `jenkins-github-gitops-credential` Secret 정리 | Not Done | canonical 계약 전환 후 유지 중 |

이 표는 재실행·장애 대응 시 사용할 Current State 기준점이며, 12 문서의 Test Case별 PASS/FAIL Evidence를 대신하지 않는다.

---

## 4. 공통 Pre-check

### 4.1 Repository 상태

```bash
cd <repo-root>
git status
git branch --show-current
git fetch --prune
git log --oneline --decorate -n 5
```

### 4.2 Ansible 실행환경

```bash
cd seokpan-infra/ansible
./ansible-safe-run playbooks/check.yml --inspect-only
.venv/bin/ansible --version            # ansible-core 2.20.8
.venv/bin/ansible-galaxy collection list kubernetes.core   # 6.5.0
.venv/bin/python -c 'import kubernetes; print(kubernetes.__version__)'  # 36.0.3
```

`cluster_kubeconfig`는 `inventory/group_vars/all/vars.yml`의 `/etc/seokpan/kubeconfig/admin.conf`를 사용한다. 개인 홈 경로 kubeconfig를 전제로 한 명령을 쓰지 않는다.

### 4.3 Argo CD Application 상태

```bash
kubectl -n argocd get applications.argoproj.io \
  -o custom-columns=NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status,REV:.status.sync.revision
```

본 영역 최소 확인 대상: `observability`, `jenkins`, `argocd-webhook-access`, `apps-backend`, `apps-frontend` 모두 `Synced / Healthy`.

### 4.4 Namespace / Workload

```bash
kubectl -n cicd get deploy,pod,pvc
kubectl -n observability get sts,deploy,ds,pod,pvc
kubectl -n observability get servicemonitor,prometheusrule
```

### 4.5 Secret 존재와 Key 이름 확인

```bash
for s in jenkins-admin harbor-robot-dockerconfig jenkins-harbor-credential \
         jenkins-harbor-api-credential jenkins-gitops-promotion-credential; do
  kubectl -n cicd get secret "$s" -o go-template='{{.metadata.name}} {{.type}}: {{range $k,$v := .data}}{{$k}} {{end}}{{"\n"}}'
done
kubectl -n observability get secret alertmanager-smtp-credential \
  -o go-template='{{range $k,$v := .data}}{{$k}}{{"\n"}}{{end}}'     # smtp-password
kubectl -n argocd get secret argocd-secret \
  -o go-template='{{range $k,$v := .data}}{{$k}}{{"\n"}}{{end}}' | grep -c 'webhook.github.secret'
```

`harbor-robot-dockerconfig`는 반드시 `Opaque` / `config.json` 키여야 한다(TS-020).

### 4.6 Harbor 상태

```bash
curl -s --cacert /etc/seokpan/pki/ca.crt https://harbor.seokpan.soldesk.store/api/v2.0/health | python3 -m json.tool
ssh harbor 'systemctl is-active harbor; systemctl is-enabled harbor'
```

모든 Component `healthy`, systemd `active / enabled`.

---

## 5. Harbor Runbook (Gate A)

### 5.1 설치·구성 적용

```bash
cd seokpan-infra/ansible
./ansible-safe-run playbooks/harbor.yml --check --diff
./ansible-safe-run playbooks/harbor.yml
```

`harbor` Role Task 구성:

```text
main.yml                 설치(offline installer, harbor_version 고정), harbor.yml 템플릿
systemd_unit.yml         VM 재부팅 후 자동 기동 (TS-010)
harbor_project.yml       seokpan Project (Private)
robot_account.yml        robot$seokpan+seokpan-ci (Push/Pull)
robot_account_api.yml    robot$seokpan+seokpan-api (API)
robot_account_runtime_pull.yml  Runtime pull-only (정태훈, harbor_runtime_pull_robot.yml)
immutability.yml         git-* 보호 / scan-** 제외 단일 selector
gc_policy.yml            GC 스케줄
```

Harbor API를 호출하는 Task는 Ansible Controller에서 `delegate_to: localhost`로 실행하고 `no_log: true`를 유지한다. 조회 Task의 결과를 후속 조건에 쓰는 경우 `check_mode: false`를 유지해 `--check`에서도 판단이 가능하도록 한다.

### 5.2 Tag Immutability 확인

```bash
./ansible-safe-run playbooks/harbor_immutability.yml --check --diff
```

API로 규칙 확인:

```bash
curl -s --cacert /etc/seokpan/pki/ca.crt -u '<api-robot>:<REDACTED>' \
  "https://harbor.seokpan.soldesk.store/api/v2.0/projects/seokpan/immutabletagrules" | python3 -m json.tool
```

기대: 활성 Rule 1개, `tag_selectors`가 `excludes: "scan-**"` **단일 항목**. `matches`와 `excludes`를 한 Rule에 함께 넣으면 저장은 `200`이지만 평가 시 예외가 무력화된다(TS-034). PUT 시 body에 URL과 같은 `id`가 없으면 `400`이 발생한다.

### 5.3 내부 CA 신뢰

```bash
./ansible-safe-run playbooks/ca_trust.yml --check --diff     # CP/Worker OS Trust Anchor
./ansible-safe-run playbooks/controller_ca_trust.yml --check --diff
```

Node에서 실제 Pull로 확인한다(`crictl` 경로).

```bash
ssh worker-01 "sudo crictl pull harbor.seokpan.soldesk.store/seokpan/frontend@<digest>"
```

### 5.4 실패 시

```text
x509 unknown authority (BuildKit Push)  → JCasC buildkit env SSL_CERT_FILE 확인 (TS-017)
x509 unknown authority (Node Pull)      → ca_trust 적용 + update-ca-trust 확인 (TS-024)
401 Unauthorized (BuildKit)             → harbor-robot-dockerconfig 형식(Opaque/config.json) (TS-020)
Robot API 404 분기 오류                  → 조회 결과 기반 분기 확인 (TS-007)
412 PRECONDITION (scan-* 삭제 거부)      → Immutability selector 구성 (TS-034)
재부팅 후 Harbor 미기동                  → systemd unit enabled 확인 (TS-010)
```

---

## 6. Jenkins Runbook (Gate B)

### 6.1 Credential 주입

```bash
cd seokpan-infra/ansible
./ansible-safe-run playbooks/jenkins_secrets.yml --check --diff
./ansible-safe-run playbooks/jenkins_secrets.yml
```

`jenkins_secrets.yml`은 `ansible-safe-run`의 RISK Playbook 목록에 포함되어 있다. Play는 `hosts: localhost / connection: local / gather_facts: false`로 특정 물리 노드에 결합되지 않는다.

주의:

- `kubernetes.io/dockerconfigjson` ↔ `Opaque` 간 type 변경은 불변 필드라 **삭제 후 재생성**이 필요하다.
- 비밀이 아닌 변수(`github_gitops_bot_username` 등)는 `inventory/group_vars/all/vars.yml`에 둔다. `group_vars/registry/vars.yml`처럼 그룹 전용 파일에 두면 `delegate_to: localhost` Task에서 `hostvars['harbor'].xxx`로만 접근된다.

### 6.2 JCasC / Manifest 변경

```text
seokpan-gitops cicd/ 변경 Branch
→ kubectl apply -f cicd/<file> --dry-run=server
→ PR (검증 로그 포함) → Merge
→ Argo CD jenkins Synced
→ JCasC 반영 확인 (Git 반영 ≠ Controller 반영)
```

JCasC ConfigMap만 바뀐 경우 Controller에 자동 반영되지 않는다. 다음 중 하나를 수행한다.

```text
Manage Jenkins → Configuration as Code → Reload existing configuration
또는
kubectl -n cicd rollout restart deploy/jenkins-controller   # Deployment spec 변경이 없을 때의 반영 수단
```

Credential 추가 시 Controller 재생성 후 Credential ID 등록을 확인한다.

```text
Manage Jenkins → Credentials → harbor-robot-account / harbor-api-robot-account / gitops-promotion-github
```

### 6.3 Plugin 변경

`jenkins-plugin-cli`는 `--plugin-file`의 확장자를 `.txt/.yaml/.yml`만 허용한다. ConfigMap 데이터 키 이름이 곧 파일명이므로 키는 반드시 `plugins-lock.txt`처럼 `.txt`로 끝나야 한다(TS-035).

Merge 전 실행 가능성 검증:

```bash
kubectl -n cicd run plugin-check --rm -it --restart=Never \
  --image=jenkins/jenkins:2.568.2-lts-jdk21@sha256:<pinned> -- \
  jenkins-plugin-cli --plugin-file /tmp/plugins-lock.txt --no-download --list
```

(ConfigMap 내용을 동일 이름 파일로 마운트해 실행한다. 내용 대조만으로 끝내지 않는다.)

### 6.4 Agent 확인

```bash
kubectl -n cicd get pod -l jenkins/label -o wide
kubectl -n cicd get pod <agent-pod> -o jsonpath='{.spec.containers[*].image}'
```

BuildKit 컨테이너 기대값:

```text
securityContext.seccompProfile.type: Unconfined
securityContext.privileged: false, runAsUser: 1000
env BUILDKITD_FLAGS=--oci-worker-no-process-sandbox
env SSL_CERT_FILE=/etc/buildkit/certs/ca.crt
env DOCKER_CONFIG=/home/user/.docker
```

### 6.5 실패 시

```text
Controller CrashLoopBackOff (Init:Error)
  → install-plugins initContainer 로그 확인
  → 원인 불명 시 즉시 git revert PR (TS-035: 19:41 Merge → 19:50 Revert)
JCasC ConfiguratorException "Item isn't a Scalar"
  → containerTemplate.args 등 Scalar 필드를 List로 쓴 경우 (TS-017 과정)
Dockerfile 1행에서 Build 중단
  → # syntax= directive 외부 frontend 재위임 (TS-028)
첫 RUN에서 /proc mount operation not permitted
  → seccomp + no-process-sandbox 조합 (TS-022)
```

---

## 7. CI Pipeline 운영 (Gate B)

### 7.1 PR Pipeline

`seokpan-app` PR 생성 시 `Jenkinsfile`이 실행된다.

```text
App CI Check (app-ci-check) → Build Verify (buildkit-rootless-pr, Push 없음)
```

PR Pipeline에서 Harbor Push / GitOps 변경 / 실제 Provider Credential이 호출되지 않는지 확인한다.

### 7.2 main Image Pipeline

`main` Merge 후 `Jenkinsfile.image-pipeline`이 실행된다. 확인 순서:

```text
1. Branch Guard: main 외 실행이면 즉시 실패해야 정상
2. P1 Revalidation PASS
3. Guard: Final Tag Check → 기존 git-<sha12> 존재 시 재사용(Scan 생략, SKIPPED_EXISTING_FINAL)
4. Build & Push Candidate: scan-<sha12>-<BUILD_NUMBER>, push-time Digest 기록
5. Scan: CRITICAL 또는 fixable HIGH → FAIL(Promote 금지)
6. Candidate Health Smoke: /health/live 200 (backend 8000 / frontend 8080)
7. Promote: git-<sha12> 부여 (HTTP 200/201)
8. Verify Digest & Cleanup Candidate: Final Digest == Candidate Digest, scan-* Tag-only 삭제
9. Evidence: image-metadata.json archive (fingerprint)
10. Create GitOps Promotion PR (8장)
```

`image-metadata.json` 확인:

```bash
curl -s -u '<user>:<REDACTED>' \
  "https://<jenkins>/job/<image-pipeline>/<BUILD_NUMBER>/artifact/image-metadata.json" | python3 -m json.tool
```

확인 필드: `commit_sha_full`, `jenkins_build_number`, `components.*.final_digest`, `scan`, `health_smoke`, `cleanup_status`, `gitops_change: DELEGATED_TO_PROMOTION_STAGE`.

### 7.3 재실행 규칙

```text
동일 Commit 재실행
→ Final Tag가 이미 immutable로 존재하면 재Build/Scan 없이 재사용
→ 새 Final Tag를 강제로 만들지 않음

Scan/Smoke 실패
→ Candidate(scan-*)는 보존 (실패 원인 분석용)
→ 원인 수정 Commit으로 새 Run (같은 Candidate 재사용 금지)
→ Harbor Retention 미구성이므로 실패 Candidate는 수동 정리 대상 (10.4 Gap)
```

`disableConcurrentBuilds()`로 동시 실행이 막혀 있으므로, 대기 중인 Build를 수동 Abort하지 않는 한 순차 실행된다.

---

## 8. GitOps Promotion / Argo CD CD Runbook (Gate C·D)

### 8.1 Promotion PR 자동 생성 확인

main Pipeline 마지막 Stage 콘솔 출력에서 결과 코드를 확인한다.

```text
PROMOTION_NO_CHANGE=1                    → 이미지 입력 경로 변경 없음, 정상 종료
PROMOTION_PR_URL=https://github.com/...  → PR 생성
PROMOTION_OPEN_PR_REQUIRES_REVIEW        → 기존 open PR 존재 — 사람이 확인 후 처리
PROMOTION_CLOSED_UNMERGED                → 과거 PR이 merge 없이 닫힘 — 사람이 확인
PROMOTION_ERROR                          → 중단, 로그 확인
```

`gitops-promotion-result.json`도 Build Artifact로 보관된다.

### 8.2 Promotion PR Review 체크리스트

```text
□ 변경 파일이 apps/<component>/kustomization.yaml, deployment.yaml 뿐인가
□ kustomization.yaml images[].digest 가 image-metadata.json final_digest 와 일치하는가
□ deployment.yaml metadata.annotations["seokpan.io/app-source-commit"] 가 App main Commit(40자)인가
□ 영향받지 않은 Component가 함께 바뀌지 않았는가
□ PR Body의 Jenkins Build URL / Scan / Health Smoke 결과가 PASS인가
```

Digest 교차 확인:

```bash
kubectl kustomize apps/frontend | grep -n 'image:'
curl -s --cacert /etc/seokpan/pki/ca.crt -u '<api-robot>:<REDACTED>' \
  "https://harbor.seokpan.soldesk.store/api/v2.0/projects/seokpan/repositories/frontend/artifacts/git-<sha12>" \
  | python3 -c 'import json,sys; print(json.load(sys.stdin)["digest"])'
```

`kustomization.yaml`이 실제 Source of Truth이며 `deployment.yaml`의 image 문자열은 placeholder다. Raw `kubectl apply -f deployment.yaml --dry-run=server`는 ArgoCD 실제 경로를 재현하지 않으므로 `kubectl apply -k apps/<component> --dry-run=server`로 검증한다.

### 8.3 Merge 후 확인

```bash
kubectl -n argocd get application apps-frontend \
  -o jsonpath='{.status.sync.status} {.status.health.status} {.status.sync.revision}{"\n"}'
kubectl -n application rollout status deploy/frontend --timeout=180s
kubectl -n application get pod -l app.kubernetes.io/name=frontend \
  -o jsonpath='{range .items[*]}{.metadata.name} {.status.containerStatuses[0].imageID}{"\n"}{end}'
```

PASS: Argo CD revision = Merge Commit, Pod `imageID`의 Digest = PR의 Digest, Ready.

`rollout status` 메시지만으로 판정하지 않고 API 수준(Argo revision, Pod imageID)까지 확인한다.

### 8.4 Promotion 수동 경로 (자동화 장애 시)

Jenkins Promotion Stage가 실패했지만 Final Digest는 확정된 경우:

```text
seokpan-gitops main에서 Branch 생성
→ kustomization.yaml digest + deployment.yaml annotation 수동 갱신 (8.2 체크리스트 동일)
→ PR → Review → Merge
```

자동화와 같은 결과물(동일 파일 두 곳)을 만들어 다음 자동 실행의 Component 영향 판정이 깨지지 않도록 annotation을 반드시 함께 갱신한다.

### 8.5 Argo CD Bootstrap 재실행

```bash
cd seokpan-infra/ansible
./ansible-safe-run playbooks/argocd_bootstrap.yml --check --diff
./ansible-safe-run playbooks/argocd_bootstrap.yml
./ansible-safe-run playbooks/argocd_bootstrap.yml   # 2회차 changed=0 확인
```

확인:

```text
argocd-application-controller StatefulSet Ready (PR #193 Validation)
argocd-cmd-params-cm server.insecure=true (PR #209)
argocd-cm resource.exclusions 에 EndpointSlice 없음 (PR #196)
kubernetes.core.k8s Task apply: true (SSA 실제 동작, PR #202)
```

argocd namespace에 GitOps로 새 리소스를 추가할 때는 Ansible 소유 리소스(argocd-cm, argocd-server 등)를 건드리지 않도록 Application 범위를 새 리소스만으로 좁힌다(`argocd-webhook-access`가 ReferenceGrant만 소유하는 이유).

### 8.6 Webhook

```bash
./ansible-safe-run playbooks/argocd_webhook_secret.yml --check --diff
./ansible-safe-run playbooks/argocd_webhook_secret.yml
```

순서가 중요하다.

```text
1. argocd-secret에 webhook.github.secret 병합 (서명 검증 활성화)
2. 그 다음 HTTPRoute 공개 (GitOps platform/gateway/base/httproute-argocd-webhook.yaml)
```

Route가 먼저 공개되면 서명 없는 Payload로 Refresh를 유발할 수 있다.

확인:

```bash
kubectl -n application get httproute argocd-webhook-route -o jsonpath='{.status.parents[*].conditions[*].type}={.status.parents[*].conditions[*].status}{"\n"}'
kubectl -n argocd get referencegrant allow-application-httproute-to-argocd-server
kubectl -n argocd logs deploy/argocd-server --since=10m | grep -i webhook
```

GitHub `seokpan-gitops` → Settings → Webhooks → Recent Deliveries에서 응답 `200`을 확인한다. 기존 `game.seokpan.soldesk.store` `/`, `/api/v1`, `/ws/v1` 경로가 영향받지 않았는지 함께 확인한다.

---

## 9. Self-Heal / Rollback Runbook (Gate C)

### 9.1 Self-Heal

Git에 선언된 필드만 Self-Heal 대상이다. Git에 없는 annotation을 추가해도 Self-Heal은 동작하지 않는다(정상).

```bash
kubectl -n application scale deploy/frontend --replicas=3
kubectl -n argocd get application apps-frontend -w     # OutOfSync → Synced
kubectl -n application get deploy frontend             # 2/2 복귀
kubectl -n application get events --sort-by=.lastTimestamp | tail
```

영구 데이터에 영향을 주는 Live Drift는 사용하지 않는다.

### 9.2 Git Revert Rollback

```text
문제 Revision 식별 (Argo revision / PR 번호)
→ git revert <merge-commit> 로 Revert PR 생성
→ Review → Merge
→ Argo CD Refresh (Webhook 또는 수동 Hard Refresh)
→ Synced / Healthy, 기존 Digest 복귀 확인
```

```bash
git switch -c revert/<pr> origin/main
git revert <squash-commit-sha>
git push -u origin revert/<pr>
# PR 생성 후 Merge
kubectl -n argocd patch application apps-frontend --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

Step 8 실측 기준선: 존재하지 않는 Digest 주입(GitOps PR #119, 10:04) → Revert Merge(GitOps PR #120, 10:15) → Deployment 2/2, Pod Digest가 Baseline(`sha256:123203a4…0b0701`)과 일치.

**관찰된 한계**: 기본 RollingUpdate가 새 Pod 검증 전에 기존 Pod를 줄여 가용 2/2 → 1/2가 됐다. `maxUnavailable: 0` 적용 여부는 Deployment Owner(정태훈) 결정 사항이다.

---

## 10. Observability Stack Runbook (Gate E)

### 10.1 구조

```text
argocd/applications/observability.yaml (Multi-source)
  ├─ Helm: prometheus-community/kube-prometheus-stack 88.5.4
  │        valueFiles: $values/observability/values-kube-prometheus-stack.yaml
  ├─ ref: values (seokpan-gitops main)
  └─ path: observability/ (Raw Manifest: Loki, Alloy, Dashboard, ServiceMonitor, NetworkPolicy, alertmanager-config)
syncOptions: ServerSideApply=true (대형 CRD 대응)
```

### 10.2 변경 절차

```text
values 또는 observability/*.yaml 수정
→ helm template 로컬 렌더링 (values 변경 시)
→ kubectl apply --dry-run=server -f observability/<file>
→ PR → Merge → Argo CD observability Synced / Healthy
→ 영향 대상 API 수준 확인 (Target, Query, Alert, Dashboard)
```

```bash
helm template kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --version 88.5.4 -n observability -f observability/values-kube-prometheus-stack.yaml > /tmp/kps.yaml
kubectl apply --dry-run=server -f /tmp/kps.yaml 2>&1 | tail
```

### 10.3 Prometheus Target 확인

```bash
kubectl -n observability port-forward svc/kube-prometheus-stack-prometheus 9090:9090 &
curl -s localhost:9090/api/v1/targets?state=active \
  | python3 -c 'import json,sys; t=json.load(sys.stdin)["data"]["activeTargets"]; \
import collections; c=collections.Counter((x["labels"]["job"],x["health"]) for x in t); \
[print(k,v) for k,v in sorted(c.items())]'
```

Down Target이 있으면 `lastError`를 먼저 본다.

```text
context deadline exceeded → NetworkPolicy Egress/Ingress 쌍 누락 가능성 우선 확인
no route to host          → 외부 방화벽 zone / 라우팅 (13장)
connection refused        → Exporter 미설치·미기동
activeTargets=0 전체      → Prometheus → kube-apiserver 6443 Egress 확인
```

controller-manager / scheduler / etcd / kube-proxy는 Issue #177 결정으로 수집하지 않는다. 해당 Job이 없는 것은 정상이다.

### 10.4 NetworkPolicy 변경

```text
1. 정책 번호·주석 패턴 유지, Egress를 추가하면 대상 측 Ingress도 같은 PR에 추가
2. DNS Egress는 kube-system으로 한정
3. 외부 IP는 /32 또는 필요한 최소 CIDR
4. Merge 후 Target / Query / Grafana Datasource로 확인
```

observability 외 namespace 정책은 소유자(Network·Kubernetes 담당) 확인 후 진행한다.

### 10.5 저장소

```bash
kubectl get pv prometheus-local-pv loki-local-pv -o wide
kubectl -n observability get pvc
ssh worker-01 'df -h /mnt/observability/prometheus'
ssh worker-02 'df -h /mnt/observability/loki'
```

Prometheus는 `worker-01`, Loki는 `worker-02` 고정이다. Deployment/StatefulSet이 참조하는 PVC 이름이 Local PV와 짝을 이루는지 먼저 확인한다(TS-029).

---

## 11. Log Runbook (Gate F)

### 11.1 Alloy 설정 변경 전 검증

`grafana/alloy:v1.6.1`에는 `validate` 서브커맨드가 없다. 문법 확인과 컴포넌트 그래프 평가를 각각 수행한다.

```bash
docker run --rm -v "$PWD/config.alloy:/c.alloy" grafana/alloy:v1.6.1 fmt -t /c.alloy
timeout 15 docker run --rm -v "$PWD/config.alloy:/c.alloy" grafana/alloy:v1.6.1 \
  run /c.alloy --server.http.listen-addr=127.0.0.1:12345 2>&1 | grep -iE 'error|evaluat'
```

### 11.2 hostPath 추가 시 SELinux Canary

노드는 SELinux enforcing이다. `runAsUser: 0`은 DAC만 만족한다. DaemonSet 변경 전 단일 Canary Pod로 확인한다.

```bash
kubectl -n observability apply -f /tmp/alloy-canary.yaml    # 동일 mount/securityContext
ssh worker-01 'sudo ausearch -m avc -ts recent | tail; sudo journalctl -k --since "-10 min" | grep -i avc'
kubectl -n observability delete -f /tmp/alloy-canary.yaml
```

### 11.3 수집 확인

```bash
kubectl -n observability get ds alloy
kubectl -n observability port-forward svc/loki 3100:3100 &
curl -s 'localhost:3100/loki/api/v1/labels' | python3 -m json.tool
curl -sG 'localhost:3100/loki/api/v1/query_range' \
  --data-urlencode 'query=sum by (node_name) (count_over_time({unit=~".+"}[5m]))' | python3 -m json.tool
curl -sG 'localhost:3100/loki/api/v1/query_range' \
  --data-urlencode 'query={namespace="application"}' --data-urlencode 'limit=5' | python3 -m json.tool
```

Loki ingress는 Grafana와 Alloy만 허용하므로 임시 Pod에서의 timeout은 정상이다. 조회는 Grafana Pod 또는 port-forward로 수행한다.

`query_range`가 `status: success`여도 `result: []`면 "조회 경로 정상"까지만 증명한다. 실제 Stream이 반환될 때 수집 PASS로 기록한다.

---

## 12. Alert / E-mail Runbook (Gate G)

### 12.1 Credential 주입과 Preflight

```bash
cd seokpan-infra/ansible
./ansible-safe-run playbooks/alertmanager_smtp_preflight.yml   # --check 미지원, 실제 실행
./ansible-safe-run playbooks/alertmanager_secret.yml --check --diff
./ansible-safe-run playbooks/alertmanager_secret.yml
```

Preflight는 전용 라벨 Job + 임시 NetworkPolicy로 DNS → TCP 587 → STARTTLS를 확인하고, 성공/실패와 무관하게 `block/always`로 Job과 임시 정책을 정리한다. 재실행 시 잔존 Job/정책을 먼저 삭제한다.

### 12.2 소유권 분리

```text
GitOps  (observability/alertmanager-config.yaml)
  smtp_smarthost, smtp_from, smtp_auth_username, smtp_auth_password_file(경로), route, receivers
Ansible (alertmanager_secret)
  alertmanager-smtp-credential Secret: smtp-password (Resend API Key)
Helm values
  alertmanagerSpec.configSecret: alertmanager-config
  alertmanagerSpec.secrets: [alertmanager-smtp-credential]
```

`configSecret`을 지정하지 않으면 Chart 기본 이름(`alertmanager-<CR>`)을 찾으므로 Git의 `alertmanager-config`와 어긋난다.

### 12.3 설정 반영 확인

```bash
kubectl -n observability exec alertmanager-kube-prometheus-stack-alertmanager-0 -c alertmanager -- \
  ls /etc/alertmanager/secrets/alertmanager-smtp-credential/
kubectl -n observability port-forward svc/kube-prometheus-stack-alertmanager 9093:9093 &
curl -s localhost:9093/api/v2/status | python3 -c 'import json,sys; print(json.load(sys.stdin)["config"]["original"])' \
  | grep -E 'smtp_smarthost|smtp_from|receiver'
```

### 12.4 End-to-End 시험 (firing → resolved)

임시 PrometheusRule로 Alert를 발생시킨다. Argo CD가 추적하지 않는 리소스이므로 시험 후 반드시 삭제한다.

```yaml
# /tmp/test-alert.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: seokpan-test-alert
  namespace: observability
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: seokpan-test
      rules:
        - alert: SeokpanTestAlert
          expr: vector(1)
          for: 0m
          labels:
            severity: warning
          annotations:
            summary: "Alertmanager E-mail E2E test"
```

```bash
kubectl apply -f /tmp/test-alert.yaml
curl -s localhost:9093/api/v2/alerts | python3 -c 'import json,sys; [print(a["labels"]["alertname"], a["status"]["state"]) for a in json.load(sys.stdin)]'
# E-mail 수신 확인 (seokpan@soldesk.store) — firing
kubectl delete -f /tmp/test-alert.yaml
# resolve_timeout 경과 후 resolved E-mail 수신 확인
```

수신 메일의 원본 헤더에서 `spf=pass`, `dkim=pass`, `dmarc=pass`를 확인한다. 스팸함 도착은 신규 도메인 평판 문제일 수 있으므로 인증 결과로 판단한다.

### 12.5 Routing 기준

```text
Watchdog / InfoInhibitor → null (메일 미발송)
severity=critical        → critical-email, repeat 30m
그 외                     → default-email, repeat 4h
send_resolved: true
```

Watchdog 메일이 다시 오기 시작하면 `match_re` Route 순서(critical보다 앞)를 확인한다.

---

## 13. 외부 VM Exporter Runbook (Gate E)

### 13.1 대상 추가 절차

```text
1. 대상 VM 소유자 확인 (Network / Data 담당)
2. Exporter 설치 (Ansible Role)
3. 로컬 /metrics 200 확인
4. 방화벽: Worker 대역(192.168.51.0/24, 192.168.52.0/24) → 포트 최소 허용
5. Worker에서 curl 확인
6. GitOps: EndpointSlice 주소 + (필요 시) NetworkPolicy 정책 3 포트 추가
7. Prometheus Target UP → Dashboard 확인
```

### 13.2 node_exporter (VRouter / Ansible Controller)

```bash
./ansible-safe-run playbooks/install_node_exporter_external.yml --check --diff
./ansible-safe-run playbooks/install_node_exporter_external.yml
./ansible-safe-run playbooks/vrouter_firewall.yml --check --diff     # external zone 9100 rich rule 포함
```

`node_exporter_linux` Role(김상희 작성)은 defaults/tasks를 수정하지 않고 재사용한다. DB·MaxScale·NFS는 `playbooks/node_exporter_linux.yml`(Data 담당)로 관리한다.

VRouter 방화벽 기준:

```text
internal zone : install_node_exporter_external.yml post_tasks 에서 9100/tcp
external zone : vrouter_firewall Role 에서 Worker 대역만 rich rule 허용
               (Prometheus 트래픽이 실제로 들어오는 인터페이스가 external zone — Issue #205)
```

단독 fix Playbook을 만들지 않는다. VRouter 재구축 경로(`vrouter_firewall`)에 포함되어야 재발하지 않는다.

확인:

```bash
ssh worker-01 'for ip in 192.168.51.10 192.168.52.10 192.168.53.10 192.168.54.10 192.168.54.70; do \
  printf "%s " $ip; curl -s -o /dev/null -w "%{http_code}\n" --max-time 3 http://$ip:9100/metrics; done'
ssh vrouter-02 'sudo firewall-cmd --zone=external --list-rich-rules'
```

2회차 실행 `changed=0`으로 멱등성을 확인한다.

### 13.3 lb01 HAProxy 네이티브 Metric

```bash
./ansible-safe-run playbooks/lb_haproxy.yml --check --diff
```

기준:

```text
frontend stats
  bind 10.1.93.78:8404        # 실IP만, VIP(10.1.93.90) 미포함
  http-request use-service prometheus-exporter if { path /metrics }
```

기존 4개 Listener(HTTP/HTTPS/K8s API/DB)와 분리되어야 한다. `haproxy -c -f /etc/haproxy/haproxy.cfg` 검증 후 reload한다. LB는 Network 담당 소유이므로 변경 전 확인한다.

```bash
ssh worker-01 'curl -s --max-time 3 http://10.1.93.78:8404/metrics | grep -m3 haproxy_'
```

### 13.4 EndpointSlice 관리 전제

Argo CD 기본 설정은 EndpointSlice를 `resource.exclusions`에 포함한다. 제외가 해제되어 있어야 Git의 EndpointSlice가 Sync된다(Infra PR #196). 새로 구축한 Argo CD에서 외부 Target이 0/0이면 이 설정부터 확인한다.

```bash
kubectl -n argocd get cm argocd-cm -o jsonpath='{.data.resource\.exclusions}' | grep -i endpointslice || echo "not excluded"
kubectl -n observability get endpointslice -l kubernetes.io/service-name=node-exporter-external -o yaml | grep -A1 addresses
```

---

## 14. Grafana Dashboard Runbook (Gate G)

### 14.1 추가·수정

```text
observability/grafana-dashboard-<name>.yaml
  metadata.labels.grafana_dashboard: "1"
  data.<name>.json: 고정 uid (seokpan-*)
→ PR → Merge → Argo CD Sync → sidecar 로드 확인
```

규칙:

- 같은 ConfigMap을 두 파일에 정의하지 않는다(ArgoCD `RepeatedResourceWarning`).
- Datasource는 `isDefault: true`가 하나만 존재해야 한다(GitOps PR #77). Prometheus가 기본이므로 패널에서 datasource uid를 하드코딩하지 않는다.
- 외부 VM / MariaDB 패널은 `and on(instance) (up{job="..."} == 1)` 조인으로 Down 인스턴스를 제외한다.
- 실제로 노출되지 않는 Metric 이름을 쓰지 않는다. 추가 전 Prometheus에서 Query로 존재를 확인한다.
- 목표값·색상 임계값은 12 문서 3.2절 근거 없이 두지 않는다.

### 14.2 로드 확인

```bash
kubectl -n observability logs deploy/kube-prometheus-stack-grafana -c grafana-sc-dashboard --since=10m | tail
kubectl -n observability port-forward svc/kube-prometheus-stack-grafana 3000:80 &
curl -s -u admin:<REDACTED> 'localhost:3000/api/search?query=' | python3 -c 'import json,sys; [print(d["uid"], d["title"]) for d in json.load(sys.stdin)]'
curl -s -u admin:<REDACTED> localhost:3000/api/datasources | python3 -c 'import json,sys; [print(d["name"], d["isDefault"]) for d in json.load(sys.stdin)]'
```

sidecar가 ConfigMap을 못 읽으면 Grafana Egress의 kube-apiserver 6443(CP 3대 /32) 규칙을 확인한다(GitOps PR #75).

---

## 15. 장애 유형별 Recovery

| 증상 | 우선 확인 | 기본 조치 |
| --- | --- | --- |
| BuildKit Push x509 | JCasC buildkit `SSL_CERT_FILE` | TS-017 기준 복원 |
| BuildKit 401 | `harbor-robot-dockerconfig` type/key | Opaque/`config.json`로 재생성(TS-020) |
| Node ImagePull x509 | Node Trust Anchor | `ca_trust` 재적용(TS-024) |
| ImagePullBackOff(not found) | GitOps Digest가 Harbor에 존재하는지 | Git Revert(9.2) |
| Jenkins Controller Init:Error | install-plugins 로그 | 즉시 git revert → 원인 수정(TS-035) |
| JCasC 변경 미반영 | Controller 재로드 여부 | Reload 또는 Controller 재생성 |
| Promotion `OPEN_PR_REQUIRES_REVIEW` | 기존 open PR 내용 | 사람이 기존 PR 처리 후 Pipeline 재실행 |
| Promotion 후 Sync 지연 | Webhook Recent Deliveries, argocd-server 로그 | 수동 Hard Refresh, Secret/Route 확인 |
| Argo CD Application OutOfSync 반복 | Ansible과 GitOps가 같은 리소스를 소유하는지 | Application 범위 축소 |
| Prometheus Target 대량 Down | NetworkPolicy 쌍 / 6443 Egress | 정책 보완 PR |
| 외부 Target `no route to host` | VRouter zone별 규칙 | `vrouter_firewall` Role로 반영 |
| 외부 Target 0/0 | Exporter 설치 여부, EndpointSlice exclusion | 설치 / argocd-cm 확인 |
| Loki CONFIG ERROR | compactor `delete_request_store` | TS-030 |
| Loki Pending | PVC 이름 ↔ Local PV | TS-029 |
| Grafana 패널 빈 화면 | sidecar 6443 Egress, 중복 isDefault | PR #75/#77 기준 |
| Alert 메일 미수신 | `/api/v2/status` 설정, Secret 마운트, Preflight | 12장 순서로 재확인 |
| Watchdog 메일 수신 | Route 순서 | null Route 복원 |

장애 복구의 공식 측정값과 Evidence는 12에서 관리한다.

---

## 16. 재실행 / 멱등성 기준

### 16.1 Ansible

- 모든 Role은 `--check --diff` → 실제 실행 → 2회차 `changed=0` 순서로 확인한다.
- 조회 결과가 후속 `assert`/조건을 결정하는 `command/shell/uri` Task는 `check_mode: false`를 유지한다.
- POST/PUT이 skip되는 `--check`에서는 그 결과를 전제로 하는 검증 Task에 `when: not ansible_check_mode`를 둔다.
- 외부 API Credential을 쓰는 Task는 `no_log: true`.
- 기존 Secret 전체를 덮어쓰지 않아야 하는 경우(argocd-secret 등)는 필드 단위 merge patch를 사용한다.
- 신규 Ansible 파일 상단에 `# 작성자 / 작성 날짜` 주석을 유지한다.

### 16.2 GitOps

- 하나의 PR에는 하나의 문제만 담는다(원인 분석 가능성).
- Merge 전 `--dry-run=server` 또는 `kubectl apply -k --dry-run=server`, Merge 후 API 수준 확인.
- Revert는 새 PR로 수행하고 기존 커밋을 강제로 덮지 않는다.

### 16.3 CI

- 동일 Commit 재실행은 기존 Final 재사용이 정상 동작이다.
- 실패 Candidate를 재사용하지 않는다.

---

## 17. Rollback Matrix

| 변경 대상 | 기본 Rollback | 주의사항 |
| --- | --- | --- |
| Harbor Role / 설정 | Ansible Git Revert 후 재실행 | Immutability Rule id/selector 정합 확인 |
| Robot Account | Role 재실행(존재 시 재생성 안 함) | Secret 값 교체 시 `jenkins_secrets` 재실행 + Controller 재생성 |
| Jenkins Manifest / JCasC / Plugin | GitOps Revert PR | selfHeal로 kubectl 수정 무효 |
| Pipeline(Jenkinsfile) | seokpan-app Revert PR | main Merge 시 Pipeline이 다시 실행됨 |
| Promotion Script | seokpan-app Revert PR | 이미 생성된 Promotion PR은 수동 Close |
| App Digest(GitOps) | Git Revert PR | Deployment 전략상 일시 가용성 저하 가능(9.2) |
| Argo CD Bootstrap | Ansible Revert 후 재실행 | argocd-secret 필드는 merge 방식 유지 |
| Webhook | GitOps Route Revert → 필요 시 Secret 제거 | Route 먼저 제거 후 Secret |
| Observability values / Manifest | GitOps Revert PR | Prometheus/Loki 데이터는 Local PV에 남음 |
| NetworkPolicy | GitOps Revert PR | Egress/Ingress 쌍 동시 Revert |
| Alertmanager 설정 | GitOps Revert PR | Credential Secret은 Ansible 소유로 별도 |
| 외부 Exporter / VRouter / LB | Ansible Revert 후 재실행 | 소유 담당자 확인 |
| Dashboard | GitOps Revert PR | UI 수정본은 복원 대상 아님 |

---

## 18. 12 검증·측정 계획으로의 인계

11은 실행 방법을 제공하지만 다음은 12의 책임이다.

```text
정식 Test Case ID (DOB-*)
Commit → Digest → PR → Sync → Ready 시간 측정
Self-Heal / Rollback 복구 시간
Target 수 / Query 결과 / Alert 수신 시각
장애 주입(F-10~F-13) Go/No-Go와 결과
Q-07 CI/CD Before/After
Run ID / Evidence Link
```

---

## 19. Runbook 완료 기준

- 실제 Repository 자산(Role/Playbook/Manifest/Jenkinsfile)과 절차가 일치한다.
- selfHeal 환경에서 Git 경로만을 영구 변경 경로로 명시한다.
- Credential은 Vault → Secret → env → JCasC 단방향 경로로만 주입한다.
- Promotion은 PR 생성까지만 자동이며 Review Gate가 유지됨을 명시한다.
- 알려진 함정(Plugin 파일 확장자, Immutability selector, JCasC 재로드, EndpointSlice exclusion, VRouter zone, SELinux hostPath, Alertmanager configSecret)을 절차에 반영한다.
- 미구현 항목(외부 VM Log, 전용 Alert Rule, Harbor Retention, Jenkins Job 선언, F-10~F-13)을 완료로 표현하지 않는다.
- 실패 시 중단·복구·Rollback 경로가 있다.
- 09와 책임 중복이 없고 12의 Test/Measurement/Evidence를 침범하지 않는다.
