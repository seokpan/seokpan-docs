[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-018 — BuildKit Harbor CA Trust 연결 중 Kubernetes 관리자 kubeconfig의 경로·소유 방식이 문제로 드러남

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 아래 내용만으로 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성했습니다.

| 항목 | 내용 |
|---|---|
| **시점** | 2026-09-01 |
| **상태** | **진행 중** |
| **주 담당** | **최유준 — CI/CD / 모니터링·관측** |
| **영향 역할** | 정태훈(Kubernetes/Argo), 이유빈(Ansible Controller Access) |
| **핵심 범주** | CA Trust / kubeconfig / Credential Boundary / Reproducibility |

## 목적

BuildKit Agent Pod가 내부 Harbor TLS를 신뢰하도록 `ca.crt`를 `cicd` Namespace Secret으로 제공하려는 작업이었다.

## 첫 번째 리뷰 문제: 새 Kubernetes Task가 명시적인 kubeconfig를 사용하지 않음

`internal_ca` Role에 새로 추가한:

- `kubernetes.core.k8s_info`
- `kubernetes.core.k8s`

Task가 `cluster_kubeconfig`를 명시하지 않았다.

Controller에서 `become`/실행 계정이 섞이는 환경에서는 기본 kubeconfig 검색 위치가 달라질 수 있다.

## 조치

두 Task에:

```yaml
kubeconfig: "{{ cluster_kubeconfig }}"
```

를 명시.

## CA/TLS 검증

- `cicd/harbor-ca-cert` Secret 생성
- CA: Seokpan Internal Root CA
- Harbor Server Certificate Issuer 일치
- CP-03 기본 Trust Store에서는:
  ```text
  Verify return code: 21
  ```
- 내부 CA를 명시:
  ```text
  Verification: OK
  Verify return code: 0 (ok)
  ```

즉 **CA Chain 자체는 정상**임을 증명했다.

## 두 번째 리뷰 문제: 공용 kubeconfig가 특정 사용자 Home에 귀속

수정 과정에서:

```yaml
cluster_kubeconfig: "/home/cyj/.kube/seokpan-admin.conf"
```

가 사용됐다.

하지만 `cluster_kubeconfig`는 `internal_ca`뿐 아니라 `argocd_bootstrap` 같은 공용 Cluster Bootstrap Role도 소비한다.

특정 개인 Home에 공용 관리자 자격증명을 고정하면:

- 다른 실행 계정 재현성 저하
- 개인 소유 자격증명과 Project Bootstrap Credential의 의미 혼합
- CA Private Key 보호 문제와 Kubernetes Admin Credential 관리 문제의 경계 혼동

이 발생한다.

## 현재 반영 방향과 실제 Main/Smoke 상태를 구분해야 함

PR #87의 변경안에서는:

```yaml
cluster_kubeconfig: "/etc/seokpan/kubeconfig/admin.conf"
```

로 공용 경로를 제안했다. 다만 PR #87은 아직 Merge되지 않았기 때문에 **현재 Main/Smoke에서 사용하는 Argo CD Role은 여전히 실행 사용자 Home 기준 경로를 참조하고 있었다.**

빈 Project Smoke에서 실제 실행 계정은 `ansible`이었고, `playbooks/argocd_bootstrap.yml`은 첫 Task에서 다음 오류로 중단됐다.

```text
Could not find or access '/home/ansible/.kube/seokpan-admin.conf'
```

추가 확인 결과:

```text
/home/ansible/.kube/seokpan-admin.conf
→ 파일 없음

/etc/seokpan/kubeconfig/admin.conf
→ ansible 계정 Permission denied
```

따라서 이 시점의 Argo CD Smoke는 **BLOCKED**였으며, Cluster나 Argo CD Runtime 장애가 아니라 Controller 실행계정 기준의 Credential 경로·권한 불일치로 판정됐다.

즉 이 문제는 단순히 “경로 문자열을 /etc로 바꾸면 끝”이 아니라 **실제 Ansible 실행계정·공용 경로·파일 권한을 동시에 맞춰야 하는 재현성 문제**로 확인됐다.

## 권한 문제에서 확정된 공용 접근 기준

`argocd_bootstrap`과 `internal_ca`의 `kubernetes.core` Task는 Controller에서 실행되므로, 실제 Playbook 실행 계정이 공용 관리자 kubeconfig를 읽을 수 있어야 한다.

따라서 `root:root 0600`만 사용하면 일반 Ansible 실행자가 파일을 읽지 못한다.

초기 확인 시에는 공용 kubeconfig가 `/etc/seokpan/kubeconfig/admin.conf`에 있어도 실제 `ansible` 실행 계정이 읽을 수 없어 `Permission denied`가 발생했다. 이에 Issue #90에서 공용 접근 기준을 다음과 같이 확정하고 실제 Ansible VM에 적용했다.

```text
directory: 750
admin.conf: 640
group: ansible-kube
```

적용된 기준은 다음과 같다.

- 공용 기준 경로: `/etc/seokpan/kubeconfig/admin.conf`
- `ansible` 실행 계정을 `ansible-kube` 그룹에 포함
- kubeconfig는 Root 소유를 유지하면서 `ansible-kube` 그룹에만 읽기 허용
- `/etc/seokpan/kubeconfig` 디렉터리 권한 `750`
- `/etc/seokpan/kubeconfig/admin.conf` 파일 권한 `640`
- 실제 `ansible` 계정에서 `ANSIBLE_READ=YES` 확인
- CA Private Key의 보관 위치와 접근 정책은 이번 Kubernetes 관리자 kubeconfig 변경과 분리

따라서 **Linux 파일 권한 문제 자체는 해결**됐다. 이후 PR #87에서 `internal_ca` Playbook을 실제 재실행해 공용 `/etc/seokpan/kubeconfig/admin.conf`를 통한 Kubernetes API 접근도 확인했다. `cicd Namespace에 Harbor CA Secret 존재 확인` Task는 `ok`, 기존 Secret 생성 Task는 `skipping` 되었고 최종 결과는 `failed=0`, `unreachable=0`, `changed=0`이었다. 즉 **공용 kubeconfig 경로 + `ansible-kube` 제한 읽기 권한 + `internal_ca`의 `cluster_kubeconfig` 소비 경로까지는 실제 실행 검증이 완료**됐다.

현재 남은 문제는 `argocd_bootstrap`까지 같은 공용 경로 기준으로 정상 동작하는지 별도로 Smoke 검증하는 것이다. PR #87과 Issue #90은 아직 Open이며, `internal_ca` 검증 완료만으로 Argo CD Bootstrap과 Jenkins BuildKit 전체 연동까지 완료됐다고 보지는 않는다.

## 반드시 구분할 것

```text
CA Private Key
→ CA 담당자의 보호 자산
→ 개인 보호 영역 가능
→ Git 금지

Kubernetes Bootstrap admin.conf
→ Cluster-scoped Bootstrap용 관리자 자격증명
→ 특정 개인 Home에 공용 기준으로 귀속하지 않음
→ Git 금지
```

## 아직 완료라고 쓰면 안 되는 이유

- PR #87은 아직 Open
- 공용 kubeconfig와 `internal_ca`의 실제 Kubernetes API 접근은 재검증 PASS했지만 `argocd_bootstrap`의 동일 기준 Smoke 완료 Evidence는 아직 없음
- 권한 조정 Issue #90은 Open 상태이며, Linux 그룹·파일 권한 및 `internal_ca` 검증은 완료됐지만 Argo CD 후속 검증과 Issue 정리가 남아 있음
- Jenkins BuildKit Agent가 실제 CA를 Mount/Trust한 뒤 Harbor Image Push까지 성공하는 전체 연동 검증은 후속 대상

## 근거

- PR #87: https://github.com/seokpan/seokpan-infra/pull/87
- 관련 Issue #79: https://github.com/seokpan/seokpan-infra/issues/79
- kubeconfig 권한 Issue #90: https://github.com/seokpan/seokpan-infra/issues/90
- #90 권한 적용 완료 Evidence: https://github.com/seokpan/seokpan-infra/issues/90#issuecomment-5491859068
- PR #87 공용 kubeconfig 실제 `internal_ca` 재실행 Evidence: https://github.com/seokpan/seokpan-infra/pull/87#issuecomment-5492170895

---
