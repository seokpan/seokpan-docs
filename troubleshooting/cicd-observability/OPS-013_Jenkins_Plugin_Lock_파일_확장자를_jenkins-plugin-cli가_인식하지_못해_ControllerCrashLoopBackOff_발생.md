# TS-035 — Jenkins Plugin Lock 파일 확장자(.lock)를 jenkins-plugin-cli가 인식하지 못해 Controller CrashLoopBackOff 발생

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-11 |
| **상태** | **해결** |
| **주 담당** | **최유준 — CI/CD 및 모니터링·관측** |
| **영향 범위** | Jenkins Controller `install-plugins` initContainer, `jenkins-controller-plugins` ConfigMap, `cicd` Namespace Jenkins Controller 가용성(재발생 시 짧은 다운타임) |

## 문제 개요

`seokpan-gitops#60`(Plugin Lock 파일 도입으로 재현성 확보)에서, `jenkins-controller-plugins` ConfigMap에 전이 의존성까지 포함한 전체 Plugin 목록을 `plugins.lock`이라는 데이터 키로 추가하고, `jenkins-controller-deployment.yaml`의 `install-plugins` initContainer가 이 파일을 참조하도록 변경한 PR을 Merge했다(19:41 KST, PR #62).

`jenkins` Application(`argocd/applications/jenkins.yaml`)이 `automated: {prune: true, selfHeal: true}`로 설정되어 있어 Merge 직후 ArgoCD가 자동으로 Sync하며 Controller Pod를 재생성했는데, 새 Pod가 `Init:Error` 상태로 반복 재시작(`BackOff`)에 빠지며 Jenkins 전체가 Down됐다.

## 원인 분석

`install-plugins` initContainer의 로그를 확인한 결과 원인은 명확했다.

```text
Unknown file type, file must have .yaml/.yml or .txt extension
```

`jenkins-plugin-cli`(Plugin Installation Manager Tool)는 `--plugin-file` 인자로 넘긴 파일의 **확장자를 검사**하며, `.txt`/`.yaml`/`.yml` 세 가지만 허용한다. 기존에 정상 동작하던 `plugins.txt`는 `.txt` 확장자였지만, 이번에 추가한 ConfigMap 데이터 키 `plugins.lock`은 파일로 마운트될 때 키 이름이 그대로 파일명이 되어 확장자가 `.lock`이 되고, `jenkins-plugin-cli`가 이를 인식하지 못해 매 실행마다 즉시 실패했다.

**Plugin 목록의 내용 자체는 문제가 없었다.** 이번 장애 이전에 변경 전 Live Jenkins의 실제 설치 상태(REST API `/pluginManager/api/json?depth=1` 캡처)와 `plugins.lock` 내용을 정적으로 대조해 일치를 확인한 상태였고, 문제는 순수하게 **파일명(확장자)** 이었다.

## 조치

### 1. 즉시 서비스 복구 (git revert)

장애 확인 직후 `git revert`로 이전 정상 커밋 상태로 되돌렸다(19:50 KST, PR #64). `kubectl edit`이나 `kubectl rollout undo` 등 클러스터를 직접 조작하는 방식은 사용하지 않았다 — `jenkins` Application이 `selfHeal: true`이므로 클러스터 직접 patch는 git 상태로 즉시 재덮어씌워져 무의미하기 때문이다.

### 2. 파일명 교체 (내용은 재사용)

ConfigMap 데이터 키를 `plugins.lock` → `plugins-lock.txt`(`.txt` 확장자)로 교체했다. 76개(전이 의존성 포함) Plugin 목록 내용은 기존과 완전히 동일하게 재사용했고, `install-plugins` initContainer의 `--plugin-file` 인자도 동일하게 갱신했다(20:10 KST, PR #65).

### 3. 재발 방지 — Merge 전 실행 가능성 사전 검증 절차 추가

이번 장애의 핵심 교훈은 "내용이 Live 상태와 일치하는가"와 "그 파일을 `jenkins-plugin-cli`가 실제로 읽을 수 있는가"가 서로 다른 검증이라는 점이다. 재적용 전에 운영 리소스와 분리된 임시 검증 절차를 추가했다.

```text
kubectl -n cicd create configmap jenkins-plugins-lock-test \
  --from-file=plugins-lock.txt=<후보 파일>

kubectl run plugin-cli-test -n cicd --restart=Never \
  --image=<jenkins-controller와 동일 이미지> \
  --overrides='{ ... jenkins-plugin-cli --plugin-file /var/jenkins_plugins/plugins-lock.txt ... }'
```

- 운영 중인 `jenkins-controller-plugins` ConfigMap과 `jenkins-controller` Deployment는 이 검증 과정에서 전혀 건드리지 않는다.
- 검증 후 테스트용 Pod/ConfigMap은 반드시 삭제한다.

## 검증

- Revert(PR #64) 적용 후 Controller Pod가 다시 `READY 1/1`로 복구됨을 확인.
- 재적용(PR #65) 전, 격리된 테스트 ConfigMap(`jenkins-plugins-lock-test`) + 테스트 Pod(`plugin-cli-test`)로 동일한 `jenkins-plugin-cli --plugin-file /var/jenkins_plugins/plugins-lock.txt` 커맨드를 사전 실행:
  - `EXIT_CODE=0`
  - `DOWNLOADED_COUNT=75` — `plugins-lock.txt`에 정의된 75개 항목과 정확히 일치
  - 로그에서 75개 Plugin 전부 `Downloaded and validated plugin X` / `Checksum valid for: X`로 정상 완료, 에러 없음
- PR #65 Merge 후 ArgoCD 자동 Sync로 재생성된 실제 Controller Pod(`jenkins-controller-86bb44c4f8-7dhmw`)가 `READY 1/1`, `RESTARTS 0`으로 정상 기동함을 확인.

## Before → After

```text
Before
jenkins-controller-plugins ConfigMap의 데이터 키: plugins.lock (확장자 .lock)
  → install-plugins initContainer가 --plugin-file plugins.lock 참조
  → jenkins-plugin-cli가 "Unknown file type, file must have .yaml/.yml or .txt extension" 에러
  → install-plugins CrashLoopBackOff
  → Jenkins Controller Down

After
데이터 키를 plugins-lock.txt(확장자 .txt)로 교체, 내용(75개 Plugin, 전이 의존성 포함)은 동일하게 재사용
  → install-plugins initContainer가 --plugin-file plugins-lock.txt 참조
  → jenkins-plugin-cli 정상 실행(EXIT_CODE=0, 75개 전부 Downloaded and validated)
  → Jenkins Controller 정상 기동(READY 1/1, RESTARTS 0)
```

## 관련 근거

- GitOps Issue #60 — Plugin Lock 파일(plugins.lock) 도입으로 재현성 확보: https://github.com/seokpan/seokpan-gitops/issues/60
- GitOps PR #62 — 최초 반영(`plugins.lock`, 확장자 문제로 장애 유발), 19:41 KST 병합: https://github.com/seokpan/seokpan-gitops/pull/62
- GitOps PR #64 — Revert, 19:50 KST 병합: https://github.com/seokpan/seokpan-gitops/pull/64
- GitOps PR #65 — 재적용(`plugins-lock.txt`로 파일명 교체, 해결), 20:10 KST 병합: https://github.com/seokpan/seokpan-gitops/pull/65
