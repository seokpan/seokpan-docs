[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-048 — Ansible Vault의 DB 비밀번호가 실제 MariaDB·Kubernetes Secret과 달라진 문제

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-10 |
| **상태** | **해결 / 값이 달라진 정확한 편집 과정은 미확정** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | Ansible Vault DB 비밀번호, MariaDB 서비스 계정, Backend·Migration DB Secret |

## 최초 문제

MariaDB 백업 작업을 확인하던 중 `game_svc`로 접속하는 과정에서 Ansible Vault에 저장된 비밀번호와 실제 MariaDB 비밀번호가 맞지 않는 것을 발견했다.

DB 관리 계정 6개를 확인한 결과, 앞서 새 비밀번호로 변경했던 세 계정만 값이 달랐다.

```text
identity_svc      MISMATCH
game_svc          MISMATCH
db_admin          MISMATCH

backup_svc        MATCH
maxscale_monitor  MATCH
repl_user         MATCH
```

## 원인 추적

PR #159 작업이 끝났을 때는 다음 세 곳에 같은 DB 비밀번호가 들어 있는 것을 확인했다.

```text
Ansible Vault
MariaDB
Kubernetes backend-db-runtime / backend-db-migration Secret
```

이 구성에서는 Ansible Vault를 DB 비밀번호의 **Source of Truth**(관리 기준으로 삼는 값)로 사용하고 있었다.

이후 `vault.yml` 변경 이력을 확인했다.

PR #159 이후 해당 파일을 변경한 작업은 PR #161 한 건이었다.

실제 비밀번호는 출력하지 않고 각 변수의 값이 바뀌었는지만 비교했다.

```text
identity_svc_password: CHANGED
game_svc_password: CHANGED
db_admin_password: CHANGED
vault_harbor_api_robot_secret: ADDED
```

반면 MariaDB 두 서버와 Kubernetes Secret은 PR #159에서 확인했던 비밀번호를 그대로 사용하고 있었다.

```text
                    identity_svc   game_svc      db_admin
mariadb-01          MATCHES_159    MATCHES_159   MATCHES_159
mariadb-02          MATCHES_159    MATCHES_159   MATCHES_159
Kubernetes Secret   MATCHES_159    MATCHES_159   MATCHES_159
현재 Vault          다른 값        다른 값        다른 값
```

즉 실제 MariaDB나 Kubernetes Secret이 바뀐 것이 아니라, Vault의 세 값만 실제 사용하는 값과 달라져 있었다.

현재 확인한 기록으로는 PR #161에서 `vault.yml`을 수정하는 과정에서 세 값이 함께 달라진 것으로 판단했다.

PR #161 작업 중 merge conflict를 해결한 기록도 있었지만, 암호화된 Vault 파일만으로는 어떤 편집이나 충돌 해결 동작에서 값이 바뀌었는지까지 확인할 수 없었다.

## 조치

MariaDB 두 서버와 Kubernetes Secret은 이미 서로 같은 값을 사용하고 있었다.

따라서 DB 비밀번호를 다시 바꾸지 않고, Ansible Vault의 세 변수만 PR #159에서 검증했던 값으로 되돌렸다.

```text
identity_svc_password
game_svc_password
db_admin_password
```

PR #161에서 새로 추가한 Harbor API Robot 비밀번호와 다른 Vault 값은 변경하지 않았다.

## 검증

복구 후 실제 비밀번호를 출력하지 않고 각 위치의 값이 같은지만 다시 확인했다.

```text
Vault 변경 범위        PASS
mariadb-01             PASS
mariadb-02             PASS
Kubernetes Secret      PASS
MaxScale 상태          PASS
Replication / GTID     PASS
비밀번호 비출력        PASS
```

Ansible도 다시 실행했다.

```text
Check Mode    changed=0 / failed=0 / unreachable=0
실제 실행     changed=0 / failed=0 / unreachable=0
재실행        changed=0 / failed=0 / unreachable=0
```

MariaDB와 Kubernetes Secret은 다시 변경하지 않았고, Vault에 저장된 세 비밀번호만 실제 사용 중인 값에 맞췄다.

## Before → After

```text
Before
MariaDB 두 서버와 Kubernetes Secret은 같은 비밀번호 사용
Ansible Vault의 세 비밀번호만 다른 값
→ 관리 기준과 실제 사용하는 값이 서로 다름

After
Vault의 세 비밀번호만 기존 검증값으로 복구
→ Vault = MariaDB-01/02 = Kubernetes Secret
→ MaxScale / Replication 상태 유지
→ 다시 실행해도 changed=0
```

## 관련 근거

- Infra Issue #169: https://github.com/seokpan/seokpan-infra/issues/169
- Infra PR #172: https://github.com/seokpan/seokpan-infra/pull/172
- PR #172 Merge Commit: `dc41791b90118fd1865d03b70cec7122dd31f95f`
- 선행 PR #159: https://github.com/seokpan/seokpan-infra/pull/159
