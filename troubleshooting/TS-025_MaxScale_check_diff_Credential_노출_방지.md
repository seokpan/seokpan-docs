[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-025 — MaxScale 설정 배포의 `--check --diff`에서 Credential 노출 위험

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경·영향·원인·조치·검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-03 ~ 2026-09-07 재검증 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | MaxScale Ansible Check Mode/Diff 출력, Console·CI Log의 Credential 노출 위험 |

## 문제 개요

MariaDB/MaxScale 계정 Credential을 Ansible Vault로 분리한 뒤에도 MaxScale 설정 배포 경로에는 별도의 노출 가능성이 남아 있었다.

MaxScale은 Runtime 설정 파일에 Monitor/Replication 계정 비밀번호를 기록해야 하므로 `maxscale.cnf` Template 자체에는 Credential이 들어간다.

당시 `ansible/roles/maxscale/tasks/configure.yml`의 Template 배포 Task에는 `diff: false`가 없었다.

따라서 다음처럼 실행하면:

```text
ansible-playbook playbooks/maxscale.yml --check --diff
```

파일 변경 Diff를 통해 `maxscale_monitor_password`, `repl_user_password` 같은 민감값이 Console 또는 CI Log에 노출될 수 있었다.

Vault에 저장했다는 사실만으로 실행 출력까지 자동 보호되는 것은 아니었다.

## 원인 분석

문제의 핵심은 Credential 저장 방식과 Ansible Diff 출력 경계가 서로 다른 보안 계층이라는 점이었다.

```text
Ansible Vault
→ Repository에 Credential 평문을 저장하지 않음

Template Rendering
→ Runtime 설정에는 실제 Credential 필요

ansible --diff
→ Render된 파일 차이를 출력 가능
→ Secret 포함 Template이면 평문 노출 위험
```

즉 Vault는 Source 저장 위험을 줄였지만, Secret이 Render된 결과 파일의 Diff까지 숨겨주지는 않는다.

## 조치

`maxscale.cnf`를 배포하는 Template Task에 다음 설정을 추가했다.

```yaml
diff: false
```

이 변경은 MaxScale 설정 내용 자체를 바꾸는 것이 아니라 해당 Task의 Diff 출력만 차단한다.

따라서 파일 내용이 실제로 바뀌지 않는 한 기존 Handler/Service 동작을 변경하지 않는다.

## 검증

초기 PR에서는 수정 자체는 Merge됐지만 수정 후 실제 출력 검증 Evidence가 충분히 남아 있지 않아 공식 Troubleshooting 게시를 보류했다.

이후 Docs Issue #53에서 프로젝트 고정 실행환경을 사용해 최소 재검증을 수행했다.

검증 기준:

```text
Project Python: 3.12.13
ansible-core: 2.20.8
```

검증 방식:

1. 현재 `seokpan-infra/main`에서 `maxscale.cnf` Template Task의 `diff: false` 반영 확인
2. Vault의 실제 `maxscale_monitor` / `repl_user` Credential을 임시 실행환경에서만 복호화
3. `ansible-playbook playbooks/maxscale.yml --check --diff` 전체 출력과 실제 Credential 값을 자동 대조
4. 실제 Credential 값은 Terminal/GitHub 문서에 기록하지 않음

결과:

- `maxscale_monitor` Credential 실제 값: 출력에서 발견되지 않음
- `repl_user` Credential 실제 값: 출력에서 발견되지 않음
- **수정 후 Credential 미노출 PASS**

따라서 본 사건의 핵심 보안 속성인 `--check --diff` 출력 경로 차단은 실제 값 기준으로 검증됐다.

## 검증 중 발견된 별도 사건

같은 재검증 실행 후반의 `maxscale --config-check`에서는 TLS cert/key `Permission denied`가 별도로 발견됐다.

확인된 상태에는 다음이 포함됐다.

```text
maxscale 사용자의 cert/key read 실패
maxscale --config-check → Permission denied
```

이 문제는 `diff: false`와 관계가 없고, MaxScale이 소비하는 TLS cert/key 권한과 공용 `tls_deploy`·MaxScale Role 사이 Desired State 정합성 문제로 분리했다.

현재 해당 사건은 [Docs Issue #63](https://github.com/seokpan/seokpan-docs/issues/63)에서 **미해결 후속 사건**으로 추적한다.

따라서 전체 `maxscale.yml`의 최종 `changed=0` 수렴 검증은 그 별도 TLS 문제 때문에 이번 재검증에서 완료하지 못했지만, 이를 Credential masking 수정 실패로 해석하지 않는다.

본 TS의 완료 Gate는 실제 Credential이 Diff 출력에서 더 이상 노출되지 않는지에 둔다.

## Before → Change → After

```text
Before
Credential을 Vault로 관리
→ maxscale.cnf에는 Runtime Credential이 Render됨
→ Template Task에 diff 보호 없음
→ --check --diff에서 민감값 노출 가능

Change
maxscale.cnf Template Task에 diff: false

After
실제 Vault Credential과 --check --diff 전체 출력 자동 대조
→ maxscale_monitor Credential 미노출
→ repl_user Credential 미노출
→ Credential Diff 노출 경로 차단 PASS
```

## 관련 사건

- [TS-018 — MaxScale TLS 적용 중 인증서 권한·상위 디렉터리·SAN 검증 문제](TS-018_MaxScale_TLS_인증서_권한_SAN_검증.md): 같은 MaxScale 영역이지만 TLS Runtime 접근 문제로 Root Cause가 다르다.
- [Docs Issue #63](https://github.com/seokpan/seokpan-docs/issues/63): #53 재검증 중 새로 발견된 현재 MaxScale TLS cert/key 권한 Drift. 아직 미해결이므로 공식 TS로 게시하지 않는다.

## 관련 근거

- Docs Issue #53: https://github.com/seokpan/seokpan-docs/issues/53
- Infra PR #104: https://github.com/seokpan/seokpan-infra/pull/104
- 관련 Infra Issue #54: https://github.com/seokpan/seokpan-infra/issues/54
- 현재 MaxScale Role: https://github.com/seokpan/seokpan-infra/tree/main/ansible/roles/maxscale
