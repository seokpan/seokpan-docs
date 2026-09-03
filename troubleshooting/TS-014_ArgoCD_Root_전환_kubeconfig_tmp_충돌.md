[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-014 — GitOps Root Application 경로 전환 중 Argo CD 초기 구성 자동화(Bootstrap) 오류가 연쇄적으로 발생

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-01 |
| **상태** | **해결** |
| **주 담당** | **정태훈(Kubernetes 플랫폼·GitOps 연동) + 최유준(Argo CD Bootstrap)** |
| **영향 범위** | 김상희(스토리지), Redis 및 애플리케이션 배포 환경 |

## 1단계 — Root 경로의 역할이 섞여 있던 문제

초기 Argo 검증용 구조에서 `apps/root`가 만들어졌다. 이후 저장소 역할을 다시 정리하면서 다음 구조가 공식 기준으로 확정됐다.

- `apps/` → Frontend/Backend 애플리케이션 실제 배포 Manifest
- `platform/` → Redis/Storage/Gateway 등 Kubernetes 플랫폼
- Argo CD 하위 Application 선언 → `argocd/applications/`

기존 Root가 `prune/selfHeal`을 사용하므로 옛 경로를 먼저 삭제하면 배포 중인 하위 Application이 함께 제거될 수 있었다.

## 안전한 전환 순서

```text
새 argocd/applications 경로 준비
→ 기존 apps/root 유지
→ Infra gitops_root_path 전환
→ Argo Root 재적용
→ Root / storage-nfs Synced·Healthy 확인
→ Storage 실행 상태 회귀검증
→ 구 apps/root 제거
```

## 잘못된 Redis 연결을 Merge 전에 폐기

Redis 실제 배포 Manifest는 `platform/redis/`에 준비돼 있었지만 초기 Redis PR #14는 하위 Application 선언을 기존 `apps/root/redis.yaml`에 추가했다.

`apps/root`가 정식 Application 선언 계층으로 확정되지 않은 상태였기 때문에 해당 PR을 Merge하지 않고 닫았다. Root 구조 전환을 먼저 완료한 뒤 `argocd/applications/`를 공식 경로로 사용하도록 Redis PR #18을 다시 구성했다.

검증 결과 Redis Application은 `Synced / Healthy`, PVC와 AOF Persistence도 정상 동작했다.

## 2단계 — kubeconfig의 `~` 경로가 자동으로 확장되지 않음

Root 경로를 전환해 `argocd_bootstrap`을 실행하는 과정에서 다음 오류가 발생했다.

```text
Could not find or access '~/.kube/seokpan-admin.conf'
on the Ansible Controller.
```

`kubernetes.core.k8s`에 전달된 `~/.kube/seokpan-admin.conf` 문자열은 Shell이 아니므로 `~`가 HOME으로 자동 확장되지 않았다.

### 조치

```yaml
{{ lookup('env', 'HOME') }}/.kube/seokpan-admin.conf
```

형태로 실제 실행 사용자의 HOME을 명시적으로 해석하도록 수정했다.

## 3단계 — `/tmp` Sticky-bit와 고정 파일명 충돌

kubeconfig 문제를 해결하고 다시 실행하자 다음 오류가 발생했다.

```text
failed final rename ... Operation not permitted
```

Role이 Root Application Manifest를 `/tmp/root-application-argocd.yaml` 고정 파일명으로 사용하고 있었다. 이전 실행에서 root 소유 파일이 남아 있으면 `/tmp`의 sticky-bit 정책 때문에 다른 실행 사용자의 Ansible Template 모듈이 해당 파일을 덮어쓸 수 없었다.

### 조치

- `ansible.builtin.tempfile`로 실행별 고유 임시 파일 생성
- Template을 임시 파일에 Render
- `kubernetes.core.k8s`가 같은 파일을 사용
- 적용 후 임시 파일 삭제

## 최종 검증

- Root Application `Synced / Healthy`
- Root Application 참조 경로 = `argocd/applications`
- `storage-nfs` `Synced / Healthy`
- NFS Provisioner `1/1 Running`
- 동일 `argocd_bootstrap` 재실행 RC=0
- 구 `apps/root` 제거 후에도 Storage 배포 상태 유지
- StorageClass `nfs-k8s` 유지
- 새 RWX 검증용 PVC `Bound`
- Pod에서 실제 파일 Write 성공 후 검증 리소스 정리

## 이 사례의 핵심

```text
저장소 디렉터리 역할 정리
→ 안전한 Root 전환 순서 설계
→ 잘못 연결된 Redis PR 폐기
→ kubeconfig 경로 해석 오류 수정
→ /tmp 파일 소유권 충돌 수정
→ Argo/Storage 회귀검증
→ 옛 경로 제거
```

하나의 경로 변경이 실제 자동화 실행과 배포 상태에 미치는 영향을 단계적으로 검증하면서 안전하게 전환한 사례다.

## 후속 운영 기준

이 보고서의 `~/.kube/seokpan-admin.conf` 오류와 당시 조치는 **2026-09-01 사건 기록**으로 그대로 보존한다. 이후 Controller의 Kubernetes 관리자 Credential 운영 기준은 추가 정합화를 거쳐 변경되었다.

현재 기준은 다음과 같다.

- Controller 공용 관리자 kubeconfig: `/etc/seokpan/kubeconfig/admin.conf`
- `cluster_kubeconfig`를 소비하는 승인된 privileged localhost Ansible 작업은 위 공용 경로를 사용
- 공용 관리자 kubeconfig 접근은 `ansible-kube` 그룹으로 제한
- 일반 Kubernetes 작업은 개인 `kubernetes-admin` 사본이 아니라 역할별 ServiceAccount/RBAC kubeconfig 사용
- 역할별 kubeconfig TokenRequest 유효기간 기준은 90일

현재 운영 기준과 후속 정합화 근거:

- `seokpan-docs` 현재 변경·결정 이력: [PROJECT_CHANGES.md — 2026-09-03 Controller 관리자 kubeconfig 및 privileged Ansible 접근 기준](../PROJECT_CHANGES.md#2026-09-03)
- `seokpan-infra` 후속 정합화 Issue #118: https://github.com/seokpan/seokpan-infra/issues/118
- 최초 공용 접근그룹 구성 이력 Issue #90: https://github.com/seokpan/seokpan-infra/issues/90

따라서 이 문서의 당시 `~/.kube/seokpan-admin.conf` 경로를 현행 경로로 치환해서 읽지 않는다. **당시 장애 원인과 해결 이력은 본문을 따르고, 현재 운영 시에는 위 후속 기준을 적용한다.**

## 관련 근거

- GitOps 구조 결정 Issue #15: https://github.com/seokpan/seokpan-gitops/issues/15
- Docs PR #16: https://github.com/seokpan/seokpan-docs/pull/16
- GitOps 경로 준비 PR #16: https://github.com/seokpan/seokpan-gitops/pull/16
- 초기 Redis 연결 PR #14(Closed/미병합): https://github.com/seokpan/seokpan-gitops/pull/14
- Root 전환 후 Redis 재구성 PR #18: https://github.com/seokpan/seokpan-gitops/pull/18
- Infra Root Path PR #81: https://github.com/seokpan/seokpan-infra/pull/81
- kubeconfig 오류 Issue #82: https://github.com/seokpan/seokpan-infra/issues/82
- kubeconfig 수정 PR #83: https://github.com/seokpan/seokpan-infra/pull/83
- `/tmp` 충돌 Issue #85: https://github.com/seokpan/seokpan-infra/issues/85
- tempfile 수정 PR #86: https://github.com/seokpan/seokpan-infra/pull/86
- 구 Root 제거 및 회귀검증 PR #17: https://github.com/seokpan/seokpan-gitops/pull/17
