[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-025 — MaxScale 설정 배포의 `--check --diff`에서 인증정보 노출 위험

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경·영향·원인·조치·검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-03 ~ 2026-09-07 재검증 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | MaxScale Ansible Check Mode/Diff 출력, Console·CI Log의 인증정보 노출 위험 |

## 문제 개요

MariaDB/MaxScale 계정 인증정보를 Ansible Vault로 분리한 뒤에도 MaxScale 설정 배포 과정에는 별도의 노출 가능성이 남아 있었다.

MaxScale은 실제 설정 파일에 Monitor/Replication 계정 비밀번호를 기록해야 하므로 `maxscale.cnf` Template에는 인증정보가 들어간다.

당시 `ansible/roles/maxscale/tasks/configure.yml`의 Template 배포 Task에는 `diff: false`가 없었다.

따라서 다음처럼 실행하면:

```text
ansible-playbook playbooks/maxscale.yml --check --diff
```

파일 변경 Diff를 통해 `maxscale_monitor_password`, `repl_user_password` 같은 민감값이 Console 또는 CI Log에 노출될 수 있었다.

Vault에 저장했다는 사실만으로 Ansible 실행 출력까지 자동으로 보호되는 것은 아니었다.

## 원인 분석

Ansible Vault가 보호하는 범위와 `--diff`가 보여주는 범위가 달랐다.

```text
Ansible Vault
→ Repository에 인증정보 평문을 저장하지 않음

Template 생성
→ 실제 maxscale.cnf에는 인증정보가 들어감

ansible --diff
→ 생성된 파일의 변경 내용을 출력할 수 있음
→ 인증정보가 포함된 Template이면 평문 노출 위험
```

즉 Vault는 저장소의 평문 노출 위험을 줄였지만, Template로 생성된 설정 파일의 Diff까지 자동으로 숨겨주지는 않는다.

## 조치

`maxscale.cnf`를 배포하는 Template Task에 다음 설정을 추가했다.

```yaml
diff: false
```

이 변경은 MaxScale 설정 내용 자체를 바꾸는 것이 아니라 해당 Task의 Diff 출력만 차단한다.

따라서 파일 내용이 실제로 바뀌지 않는 한 기존 Handler나 Service 동작에는 영향을 주지 않는다.

## 검증

초기 PR에서는 수정 자체는 Merge됐지만 수정 후 실제 출력 검증 기록이 충분하지 않아 공식 TS 게시를 보류했다.

이후 Docs Issue #53에서 프로젝트 고정 실행환경을 사용해 최소 재검증을 수행했다.

검증 기준:

```text
Project Python: 3.12.13
ansible-core: 2.20.8
```

검증 방식:

1. 현재 `seokpan-infra/main`에서 `maxscale.cnf` Template Task의 `diff: false` 반영 확인
2. Vault의 실제 `maxscale_monitor` / `repl_user` 인증정보를 임시 실행환경에서만 복호화
3. `ansible-playbook playbooks/maxscale.yml --check --diff` 전체 출력과 실제 인증정보 값을 자동 대조
4. 실제 인증정보 값은 Terminal이나 GitHub 문서에 기록하지 않음

결과:

- `maxscale_monitor` 실제 인증정보: 출력에서 발견되지 않음
- `repl_user` 실제 인증정보: 출력에서 발견되지 않음
- **수정 후 인증정보 미노출 PASS**

따라서 본 사건의 핵심 검증 항목인 `--check --diff` 출력에서 인증정보가 보이지 않는 것을 실제 값 기준으로 확인했다.

## 검증 중 발견된 별도 사건

같은 재검증 실행 후반의 `maxscale --config-check`에서는 TLS 인증서와 개인키 `Permission denied`가 별도로 발견됐다.

확인된 상태에는 다음이 포함됐다.

```text
maxscale 사용자의 cert/key read 실패
maxscale --config-check → Permission denied
```

이 문제는 `diff: false`와 관계가 없고, 공용 `tls_deploy`와 MaxScale Role이 같은 TLS 파일에 서로 다른 권한을 적용하는 문제로 분리했다.

TS-025를 재검증하던 당시에는 이 문제가 해결되지 않은 상태여서 [Docs Issue #63](https://github.com/seokpan/seokpan-docs/issues/63)에서 별도로 추적했다. 이후 원인과 재발 조건을 확인해 수정·재검증까지 완료했으며, 현재는 [TS-026 — 공용 TLS Role과 MaxScale Role의 권한 설정 충돌로 인증서 접근 권한이 다시 사라짐](TS-026_MaxScale_TLS_권한_Desired_State_충돌_Drift.md)으로 기록되어 있다.

따라서 당시 전체 `maxscale.yml`의 최종 `changed=0` 확인은 별도 TLS 권한 문제 때문에 완료하지 못했지만, 이를 `diff: false` 수정 실패로 해석하지 않는다.

본 TS의 완료 기준은 **실제 인증정보가 `--check --diff` 출력에서 더 이상 노출되지 않는지**였다.

## Before → Change → After

```text
Before
인증정보를 Vault로 관리
→ maxscale.cnf에는 실제 인증정보가 들어감
→ Template Task에 diff 보호 없음
→ --check --diff에서 민감값 노출 가능

Change
maxscale.cnf Template Task에 diff: false

After
실제 Vault 인증정보와 --check --diff 전체 출력 자동 대조
→ maxscale_monitor 인증정보 미노출
→ repl_user 인증정보 미노출
→ Diff 출력에서 인증정보가 보이지 않음을 확인
```

## 관련 사건

- [TS-018 — MaxScale TLS 적용 중 인증서 파일 권한과 SAN 검증 문제](TS-018_MaxScale_TLS_인증서_권한_SAN_검증.md): 같은 MaxScale 영역이지만 최초 TLS 적용 당시의 파일 권한·SAN 문제로 원인이 다르다.
- [TS-026 — 공용 TLS Role과 MaxScale Role의 권한 설정 충돌로 인증서 접근 권한이 다시 사라짐](TS-026_MaxScale_TLS_권한_Desired_State_충돌_Drift.md): TS-025 재검증 중 발견됐던 후속 TLS 권한 문제가 이후 해결·게시된 사례다.

## 관련 근거

- Docs Issue #53: https://github.com/seokpan/seokpan-docs/issues/53
- Infra PR #104: https://github.com/seokpan/seokpan-infra/pull/104
- 관련 Infra Issue #54: https://github.com/seokpan/seokpan-infra/issues/54
- 후속 TLS 권한 문제 Docs Issue #63: https://github.com/seokpan/seokpan-docs/issues/63
- 현재 MaxScale Role: https://github.com/seokpan/seokpan-infra/tree/main/ansible/roles/maxscale
