[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-018 — BuildKit Harbor CA Trust 연결 중 Kubernetes 관리자 kubeconfig의 경로·소유 방식이 문제로 드러남

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치, 검증 결과와 남은 사항을 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-01 |
| **상태** | **진행 중** |
| **주 담당** | **최유준 — CI/CD 및 모니터링·관측** |
| **영향 범위** | 정태훈(Kubernetes 플랫폼·Argo CD 연동), 이유빈(Ansible Controller Access) |

## 문제 개요

목적은 BuildKit Agent가 Harbor의 Internal CA를 신뢰하게 만드는 것이었다.

Ansible `internal_ca` Role에서 공개 CA 인증서를:

```text
cicd/harbor-ca-cert Secret
```

으로 배포하도록 PR #87이 작성됐다.

CA Private Key는 포함하지 않았다.

## 첫 번째 리뷰 문제: 새 Kubernetes Task가 명시적인 kubeconfig를 사용하지 않음

새로운:

```text
kubernetes.core.k8s_info
kubernetes.core.k8s
```

Task가 Controller에서 실행되지만 kubeconfig를 명시하지 않았다.

따라서 실행 사용자/`become` 조건에 따라 엉뚱한 kubeconfig를 찾을 수 있었다.

## 조치

두 Task에:

```yaml
kubeconfig: "{{ cluster_kubeconfig }}"
```

를 적용했다.

## CA/TLS 검증

후속 검증에서:

- `cicd/harbor-ca-cert` 존재
- CA = `Seokpan Internal Root CA`
- Harbor 인증서 Issuer 일치
- 기본 Trust Store에서는:

```text
Verify return code: 21
```

- CAfile을 명시하면:

```text
Verification: OK
Verify return code: 0
```

즉 CA → Harbor Server Cert 신뢰 관계는 정상.

## 두 번째 리뷰 문제: 공용 kubeconfig가 특정 사용자 Home에 귀속

수정 과정에서 공용 변수:

```text
cluster_kubeconfig
```

가:

```text
/home/cyj/.kube/seokpan-admin.conf
```

로 지정됐다.

하지만 이 값은:

- `internal_ca`
- `argocd_bootstrap`
- 이후 Kubernetes Cluster-scoped Automation

이 공통으로 소비한다.

즉 특정 팀원의 Home은 공용 초기 구성 자동화(Bootstrap) Credential 위치로 적절하지 않다.

## 현재 반영 방향과 실제 Main/Smoke 상태를 구분해야 함

이 검토 이후 프로젝트 공용 기준은:

```text
/etc/seokpan/kubeconfig/admin.conf
```

로 확정했다.

또한:

```text
root:ansible-kube
Directory 0750
File 0640
```

처럼 제한된 그룹만 읽도록 정했다.

다만 이 내용은 **설계 방향이 확정됐다는 사실과 실제 Runtime에 모두 적용·검증됐다는 사실을 분리**해서 봐야 한다.

최신 빈 Project 최소 동작 검증(Smoke Test)에서 확인한 실제 상태는 다음과 같다.

```text
실행 계정: ansible

/home/ansible/.kube/seokpan-admin.conf
→ 파일 없음

/etc/seokpan/kubeconfig/admin.conf
→ 최초 Smoke 시 Permission denied
```

따라서 당시 `argocd_bootstrap`은 첫 Kubernetes Task에서 **BLOCKED**됐다.

후속 Issue #90에서는 실제 Ansible Controller에서 다음 접근 권한까지 적용됐다.

```text
ansible-kube 그룹 존재
ansible 사용자 → ansible-kube 포함
/etc/seokpan/kubeconfig → group ansible-kube / 0750
/etc/seokpan/kubeconfig/admin.conf → group ansible-kube / 0640
ANSIBLE_READ=YES
```

그리고 PR #87의 후속 실행에서는 공용 kubeconfig와 `ansible-kube` 권한을 사용해 `internal_ca`를 실제 재실행했다.

```text
cicd Namespace에 Harbor CA Secret 존재 확인 → ok
Harbor CA Secret 생성 → 기존 Secret이 있어 skipping
failed=0
unreachable=0
changed=0
```

즉 **`internal_ca`가 `/etc/seokpan/kubeconfig/admin.conf`를 읽어 Kubernetes API에 접근하는 경로는 실제 검증까지 완료**됐다.

다만 `argocd_bootstrap`은 기존:

```text
/home/ansible/.kube/seokpan-admin.conf
```

참조가 남아 있다고 #90에서 확인됐으므로, Role을 공용 `cluster_kubeconfig` 기준으로 맞춘 뒤 별도 Smoke 재실행이 여전히 필요하다.

## 권한 문제에서 확정된 공용 접근 기준

여기서 또 하나 중요한 실행상의 조건이 있었다.

처음에는:

```text
root:root 0600
```

처럼 관리자만 읽는 방안도 생각할 수 있었다.

그러나 현재 프로젝트 Ansible 실행 방식에서 `kubernetes.core` Task는 Controller에서:

```text
delegate_to: localhost
become: false
```

로 실행된다.

따라서 root-only 파일은 현재 구조와 충돌한다.

프로젝트에서 채택한 현실적 기준:

```text
/etc/seokpan/kubeconfig/admin.conf
root:ansible-kube
0640
```

Directory:

```text
/etc/seokpan/kubeconfig
root:ansible-kube
0750
```

즉:

- 일반 작업자는 읽지 못함
- 승인된 초기 구성 실행자만 `ansible-kube` Group
- 공용 Role은 동일 변수 사용
- Credential 내용은 Git에 저장하지 않음

## 반드시 구분할 것

**CA Private Key**

```text
인증서 서명용
최유준 보호 영역
이번 변경에서 이동하지 않음
```

**Kubernetes 관리자 kubeconfig**

```text
Cluster Bootstrap Credential
특정 팀원 개인 Credential이 아님
공용 자동화가 사용
```

둘은 보안 자산이지만 목적이 다르다.

## 아직 완료라고 쓰면 안 되는 이유

- PR #87은 아직 Open
- `/etc/seokpan/kubeconfig/admin.conf` Group/Permission 적용 완료
- `internal_ca`의 공용 kubeconfig 기반 Kubernetes API 접근 및 기존 `harbor-ca-cert` 확인은 실제 재실행 완료
- `argocd_bootstrap`의 공용 `cluster_kubeconfig` 경로 정합화 및 Smoke 재실행 완료 Evidence는 아직 없음
- 권한 조정 Issue #90은 Open 상태이며, Linux 그룹·파일 권한 및 `internal_ca` 검증은 완료됐지만 Argo CD 후속 검증과 Issue 정리가 남아 있음
- Jenkins BuildKit Agent가 실제 CA를 Mount/Trust한 뒤 Harbor Image Push까지 성공하는 전체 연동 검증은 후속 대상

## 관련 근거

- PR #87: https://github.com/seokpan/seokpan-infra/pull/87
- 관련 Issue #79: https://github.com/seokpan/seokpan-infra/issues/79
- kubeconfig 권한 Issue #90: https://github.com/seokpan/seokpan-infra/issues/90
- #90 권한 적용 완료 Evidence: https://github.com/seokpan/seokpan-infra/issues/90#issuecomment-5491859068
- PR #87 공용 kubeconfig 실제 `internal_ca` 재실행 Evidence: https://github.com/seokpan/seokpan-infra/pull/87#issuecomment-5492170895
