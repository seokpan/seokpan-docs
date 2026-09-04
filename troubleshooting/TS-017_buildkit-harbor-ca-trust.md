# Jenkins Rootless BuildKit Harbor CA Trust 미적용 문제

## 발생/발견 시기
2026-09-02 ~ 2026-09-03

## 담당 역할
최유준 (Delivery & Observability)

## 영향 범위
- `seokpan-infra` / `seokpan-gitops` 레포
- Jenkins Rootless BuildKit Agent Pod (`cicd` 네임스페이스)의 모든 Harbor Push 작업
- CI/CD 파이프라인(Build → Push → GitOps) 경로 전체가 차단된 상태였음 — 다른 서비스(게임 서비스 자체)에는 영향 없음

## 증상

`ci-verify-buildkit-push` 검증 Job에서 Harbor Push 시도 시 다음 에러로 반복 실패:

failed to fetch anonymous token: Get "https://harbor.seokpan.soldesk.store/service/token?...":
tls: failed to verify certificate: x509: certificate signed by unknown authority

`harbor-ca-cert` ConfigMap(Harbor 내부 CA)이 Agent Pod에 정상 마운트되어 있었고, `curl --cacert`로 직접 테스트 시 TLS 검증이 정상 통과함에도 buildctl을 통한 Push에서만 지속적으로 동일 에러가 발생했다.

## 원인

### 1차 오판 — buildkitd.toml 미적용으로 추정
JCasC PodTemplate의 buildkit container `args`에 `--config=/home/user/.config/buildkit/buildkitd.toml`을 지정하면 해결될 것으로 예상했으나, `args`를 YAML list(`- "--config=..."`)로 작성하여 JCasC ConfiguratorException(`Item isn't a Scalar`)이 발생, Controller가 CrashLoopBackOff에 빠짐. Scalar 문자열로 수정 후 정상 기동은 됐으나 TLS 에러는 그대로 재현됨.

### 2차 오판 — daemon 소켓 충돌 의심
컨테이너 시작 시 뜨는 지속형 buildkitd(container args로 기동)와, Jenkinsfile이 호출하는 `buildctl-daemonless.sh`가 매번 새로 띄우는 ephemeral buildkitd가 같은 기본 소켓 주소를 두고 경합할 가능성을 의심. `command: "cat"` + `ttyEnabled: true`로 지속형 daemon을 완전히 제거했으나, 여전히 동일한 TLS 에러가 재현되어 이 가설은 기각됨.

### 최종 확인된 근본 원인
`buildctl-daemonless.sh`를 통해 실행되는 `buildctl` 클라이언트는 Harbor의 anonymous token 발급 요청을, buildkitd 데몬이 아니라 **클라이언트 프로세스 자체의 client-side 인증 경로**(`session/auth/authprovider.(*authProvider).FetchToken`)를 통해 수행한다. 이 경로는 `buildkitd.toml`의 `[registry."<host>"] ca=[...]` 설정을 전혀 참조하지 않고, Go 표준 라이브러리(`crypto/x509`)의 시스템 CA 로딩 방식(`SSL_CERT_FILE` 환경변수)만을 따른다. `buildkitd.toml`은 buildkitd 데몬 자신이 registry와 직접 통신할 때만 유효했다.

이 원인은 buildkitd를 daemon args 없이 foreground로 직접 실행하고 `--debug` 로그를 확보하여, 실패 지점의 Go 스택 트레이스에서 `authprovider` 패키지를 확인함으로써 규명했다.

## 조치

1. JCasC `buildkit` container의 `envVars`에 `SSL_CERT_FILE=/etc/buildkit/certs/ca.crt` 추가
2. 불필요해진 `BUILDKITD_CONFIG` 환경변수, container `args`(`--config=...`) 제거
3. `command: "cat"` + `ttyEnabled: true`로 컨테이너를 대기 상태로 유지, 실제 buildkitd는 `buildctl-daemonless.sh` 호출 시점에만 기동하도록 정리 (원인은 아니었으나 구조적으로 더 안전한 방식으로 채택)
4. Jenkinsfile(`ci-verify-buildkit-push`)의 Build & Push Stage에도 동일하게 `export SSL_CERT_FILE=/etc/buildkit/certs/ca.crt` 반영

### 부수적으로 발견·해결한 2차 문제

TLS 문제 해결 직후 `401 Unauthorized`가 새로 발생. `harbor-robot-dockerconfig` Secret이 `kubernetes.io/dockerconfigjson` 타입이라 data key가 `.dockerconfigjson`으로 고정되는데, buildctl은 `DOCKER_CONFIG` 디렉토리에서 정확히 `config.json` 파일명만 인식하여 credential을 전혀 읽지 못하고 있었음. `seokpan-infra`의 `jenkins_secrets` Ansible Role에서 Secret을 `type: Opaque`, key `config.json`으로 재생성하여 해결. (Kubernetes Secret의 `type` 필드는 불변이라 delete 후 재생성 필요했음.)

## 검증 결과

- `kubectl exec ... -- ps aux`/`cat /proc/<pid>/cmdline`으로 buildkitd 실행 커맨드 직접 확인
- 수동 foreground buildkitd 재현 테스트로 `SSL_CERT_FILE` 적용 전/후 에러 메시지 변화 확인 (`x509: unknown authority` → `401 Unauthorized` → 정상 성공)
- Secret 재생성 후 `/home/user/.docker/config.json` 파일 실제 생성 확인
- `ci-verify-buildkit-push` Job 재실행, TLS/인증 에러 없이 Harbor Push 완전 성공
- Harbor UI에서 해당 시각 기준 `seokpan/test-push` 저장소에 신규 Tag+Digest 등록 확인

## GitHub 근거

- seokpan-gitops PR #19: BuildKit Agent 안정성·재현성 및 Harbor CA Trust 정합화
- seokpan-infra: harbor-robot-dockerconfig Secret 타입 변경 PR (Opaque/config.json)
- Jenkins 매니페스트 갭 5건 Issue (항목 2: Harbor CA Trust Store)

## Key Learnings

- `buildctl-daemonless.sh`는 `BUILDKITD_CONFIG`/`--config` 플래그가 아니라, client-side 인증 경로에서는 Go 표준 `SSL_CERT_FILE` 환경변수를 통해서만 커스텀 CA를 인식한다.
- JCasC `containerTemplate.args`는 YAML list가 아니라 Scalar 문자열이어야 한다.
- Kubernetes Secret의 `type` 필드는 불변이며, 타입 변경 시 patch가 아니라 delete 후 재생성이 필요하다.
- 클라이언트와 데몬이 분리된 도구(buildctl/buildkitd)에서는 "설정 파일을 어디에 마운트했는가"보다 "그 설정을 실제로 어느 프로세스/코드 경로가 읽는가"를 먼저 확인해야 한다.

