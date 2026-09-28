[← 전체 트러블슈팅](../README.md) · [CI/CD·관측 목차](README.md)

# OPS-015 — Jenkins Location URL 미설정으로 `BUILD_URL`을 생성하지 못한 문제

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-11 |
| **상태** | **해결** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | Jenkins JCasC Location 설정, Build URL 생성, CI/CD 실패 경로 검증 및 후속 Promotion Workflow |

## 최초 문제

GitOps Issue #61과 PR #63에는 A-09 실패 경로 시험의 첫 Jenkins Build가 다음 상태로 중단됐다고 기록되어 있다.

```text
Build URL missing
```

당시 Jenkins Kubernetes Cloud에는 Agent가 Controller에 접근할 때 사용하는 `jenkinsUrl`이 이미 존재했다. 그러나 이 값은 Jenkins가 Job/Build 링크를 생성할 때 사용하는 **Jenkins Location URL**과는 다른 설정이다.

이번 재검토에서 연결된 App Issue #58의 원 댓글은 실패 시험의 준비 범위만 확인됐고 위 오류 문자열 자체는 포함하지 않았다. 따라서 이 문서에서 최초 실패 문자열의 직접 근거는 GitOps Issue #61과 PR #63의 사건 기록으로 한정한다.

## 원인

Jenkins Configuration as Code(JCasC)에 Jenkins Location URL이 정의되어 있지 않았다.

기존 구성에는 Kubernetes Cloud의 Controller 접속 정보는 있었지만 다음 설정이 없었다.

```yaml
unclassified:
  location:
    url: ...
```

사건의 핵심은 다음과 같다.

```text
Kubernetes Cloud jenkinsUrl 존재
→ Agent가 Controller에 접속할 주소는 존재

Jenkins Location URL 미설정
→ Jenkins가 Job/Build의 기준 URL을 생성할 정보 부족
→ 실패 경로 시험에서 BUILD_URL missing
```

따라서 Kubernetes Cloud의 `jenkinsUrl`과 Jenkins Location은 같은 값을 사용할 수 있더라도 책임이 다른 설정으로 구분해야 했다.

Jenkins 공식 자료도 이 구분을 뒷받침한다.

- Jenkins Pipeline 문서는 `BUILD_URL`을 현재 Build 결과의 URL로 설명하고, `JENKINS_URL`은 Jenkins URL이 System Configuration에 설정된 경우에만 제공된다고 설명한다.
- Jenkins Core의 `JenkinsLocationConfiguration`은 Jenkins의 HTTP URL을 저장하는 전역 설정이다.
- Jenkins Kubernetes Plugin은 Cloud의 Jenkins URL을 해당 Cloud에서 시작한 Agent가 Controller에 연결할 때 사용하는 URL로 설명한다.

이 공식 자료는 프로젝트의 실제 장애 Evidence를 대신하지 않으며, #61/#63에서 구분한 두 설정의 기술적 역할을 교차검증하는 근거로만 사용한다.

## 조치

GitOps PR #63에서 `cicd/jenkins-jcasc-configmap.yaml`의 JCasC 설정에 Jenkins Location을 추가했다.

```yaml
unclassified:
  location:
    url: "http://jenkins-controller.cicd.svc.cluster.local:8080/"
```

당시 프로젝트에서는 Jenkins의 별도 공식 hostname을 사용하지 않았으므로 기존 내부 Service 주소를 기준 URL로 사용했다.

변경 범위는 Jenkins Location 한 항목으로 제한했다.

변경하지 않은 항목:

- Jenkins Kubernetes Cloud URL
- Service 노출 방식
- TLS
- Credential
- RBAC
- Agent 구성
- 기존 Build 기록

병합 전에 설치 중인 Jenkins/JCasC 버전에서 적용 후보와 복구 후보를 각각 검사했다.

```text
Jenkins 2.568.2
JCasC 2121.v86fe99d4b_b_a_b_

/manage/configuration-as-code/check
→ 적용 후보 HTTP 200 / []
→ 복구 후보 HTTP 200 / []
```

Argo CD가 관리하는 ConfigMap과 Controller에서 소비하는 설정을 구분하기 위해 Git → live ConfigMap → Controller Mount를 대조하고, 이미 Controller 재생성으로 설정이 반영된 경우에는 불필요한 reload를 반복하지 않는 절차를 사용했다.

## 검증

PR #63 병합 자체는 Runtime 완료로 판정하지 않았다.

초기 후보 검증 시점에는 실제 Jenkins 저장값과 후속 Build의 `BUILD_URL` 확인이 남아 있었기 때문에 GitOps Issue #61을 계속 Open 상태로 유지했다.

이후 2026-09-23 실제 Jenkins/Image Promotion Workflow에서 후속 Runtime Evidence를 다시 확인했다.

```text
GitOps Promotion PR #132
→ Jenkins Build #21
→ BUILD_URL 정상 생성

GitOps Promotion PR #133
→ Jenkins Build #22
→ BUILD_URL 정상 생성

GitOps Promotion PR #134
→ Jenkins Build #23
→ BUILD_URL 정상 생성
```

세 Build 모두 다음 형태의 Jenkins 내부 Location 기반 URL을 기록했다.

```text
http://jenkins-controller.cicd.svc.cluster.local:8080/job/.../<build>/
```

PR #132, #133, #134의 본문에는 각각 Jenkins Build #21, #22, #23과 내부 Service 주소를 기준으로 한 Build URL이 기록되어 있다. 세 PR은 모두 병합됐다.

따라서 후속 Promotion 흐름에서 Jenkins가 `BUILD_URL`을 생성하고, 그 값이 Promotion PR 본문까지 전달되는 경로를 반복 확인했다. 이 결과를 전체 Jenkins 기능 또는 모든 CI/CD 실패 경로의 정상화로 확대하지 않는다.

별도 복구 실행은 필요 상황이 발생하지 않아 수행하지 않았다.

## Before → After

```text
Before
Kubernetes Cloud jenkinsUrl 존재
+ Jenkins Location URL 미설정
→ Agent 연결 설정은 존재
→ Jenkins BUILD_URL 생성 기준은 없음
→ A-09 실패 경로 시험에서 Build URL missing

After
JCasC unclassified.location.url 추가
→ Jenkins 내부 Service URL을 Location으로 사용
→ 후속 Build #21 / #22 / #23에서 BUILD_URL 반복 생성
→ Job/Agent 기반 Promotion Workflow 정상 수행
```

## 운영 제약

Location에는 Kubernetes 내부 Service 주소를 사용하므로 Windows Host에서 해당 주소를 직접 열 수 없다.

당시 운영에서는 기존 localhost tunnel을 통해 동일한 Job/Build 경로에 접근했다. 이는 `BUILD_URL` 생성 실패와는 별개의 접근 제약이며, Issue #61 Closeout에서는 기능 미완료로 보지 않았다.

또한 같은 시기에 진행된 Jenkins Plugin Lock 문제는 Controller 재생성을 포함할 수 있어 작업 시간을 조율했지만, Plugin Lock 파일 문제는 본 사건의 Root Cause가 아니다.

## 관련 근거

- GitOps Issue #61 — Jenkins 내부 Location URL 설정 및 적용·복구 절차: https://github.com/seokpan/seokpan-gitops/issues/61
- Issue #61 적용·복구 후보 검사: https://github.com/seokpan/seokpan-gitops/issues/61#issuecomment-5632650222
- GitOps PR #63 — Jenkins Location에 내부 Service URL 지정: https://github.com/seokpan/seokpan-gitops/pull/63
- PR #63 Merge Commit: `1d8c53dafc2f4870b723a6a407855fd4bdbe81b8`
- Issue #61 Closeout Runtime Evidence: https://github.com/seokpan/seokpan-gitops/issues/61#issuecomment-5791840170
- GitOps Promotion PR #132: https://github.com/seokpan/seokpan-gitops/pull/132
- GitOps Promotion PR #133: https://github.com/seokpan/seokpan-gitops/pull/133
- GitOps Promotion PR #134: https://github.com/seokpan/seokpan-gitops/pull/134
- Jenkins Pipeline 환경변수 문서: https://www.jenkins.io/doc/book/pipeline/jenkinsfile/#using-environment-variables
- Jenkins Core `JenkinsLocationConfiguration` Javadoc: https://javadoc.jenkins.io/jenkins/model/JenkinsLocationConfiguration.html
- Jenkins Kubernetes Plugin 문서: https://github.com/jenkinsci/kubernetes-plugin
