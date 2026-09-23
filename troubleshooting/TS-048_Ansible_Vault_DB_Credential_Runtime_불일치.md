[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-048 — Ansible Vault의 DB Credential이 MariaDB·Kubernetes Secret과 달라진 문제

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-10 |
| **상태** | **해결 / 정확한 개별 충돌 해결 동작은 미확정** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | Ansible Vault DB Credential, MariaDB 서비스 계정, Backend Runtime·Migration Secret |

## 최초 문제

MariaDB 백업 검증 과정에서 `game_svc` 계정 접속을 확인하다가 현재 Ansible Vault의 `game_svc_password`와 실제 MariaDB 계정 정보가 일치하지 않는 것을 발견했다.

DB 관리 계정 6개를 전체 확인한 결과 PR #159에서 회전했던 세 계정만 불일치했다.

```text
identity_svc      MISMATCH
game_svc          MISMATCH
db_admin          MISMATCH

backup_svc        MATCH
maxscale_monitor  MATCH
repl_user         MATCH
```

## 원인 추적

PR #159 완료 시점에는 다음 세 위치가 같은 Credential을 사용하도록 검증돼 있었다.

```text
Ansible Vault
MariaDB
Kubernetes backend-db-runtime / backend-db-migration Secret
```

PR #159 이후 `vault.yml` 변경 이력을 추적한 결과 다음 변경은 PR #161 한 건에서 확인됐다.

실제 값은 출력하지 않고 변수별 변경 여부만 비교했다.

```text
identity_svc_password: CHANGED
game_svc_password: CHANGED
db_admin_password: CHANGED
vault_harbor_api_robot_secret: ADDED
```

반면 MariaDB 양 노드와 Kubernetes Secret은 PR #159에서 검증된 Credential을 그대로 유지하고 있었다.

```text
                    identity_svc   game_svc      db_admin
mariadb-01          MATCHES_159    MATCHES_159   MATCHES_159
mariadb-02          MATCHES_159    MATCHES_159   MATCHES_159
Kubernetes Secret   MATCHES_159    MATCHES_159   MATCHES_159
현재 Vault          다른 값        다른 값        다른 값
```

따라서 실제 Runtime이 변경된 것이 아니라 Source of Truth로 관리하려던 Vault의 세 값만 현재 상태에서 이탈한 것으로 확인했다.

현재 Evidence 기준으로는 PR #161의 `vault.yml` 변경 과정에서 DB Credential 3종이 의도하지 않게 다른 값으로 변경된 회귀로 판단했다.

다만 암호화된 Vault 파일의 Git 기록만으로 정확히 어떤 개별 편집 또는 merge conflict 해결 명령에서 값이 바뀌었는지까지는 확정하지 않는다.

## 조치

MariaDB와 Kubernetes Secret은 서로 일치하며 이미 검증된 Credential을 사용하고 있었기 때문에 DB Password를 다시 회전하지 않았다.

PR #159 Merge 시점의 검증된 값을 기준으로 현재 Vault의 다음 세 변수만 복구했다.

```text
identity_svc_password
game_svc_password
db_admin_password
```

PR #161에서 추가된 Harbor API Robot Secret과 다른 Vault 값은 그대로 유지했다.

## 검증

복구 후 실제 값은 출력하지 않고 변경 범위와 Runtime 정합성을 다시 확인했다.

```text
Vault 변경 범위        PASS
mariadb-01 정합성      PASS
mariadb-02 정합성      PASS
Kubernetes Secret      PASS
MaxScale 상태          PASS
Replication/GTID       PASS
Secret 값 비노출       PASS
```

Ansible 실행도 다음과 같이 수렴했다.

```text
Check Mode       changed=0 / failed=0 / unreachable=0
Actual Run       changed=0 / failed=0 / unreachable=0
Idempotency Run  changed=0 / failed=0 / unreachable=0
```

따라서 실제 DB와 Kubernetes Secret을 다시 변경하지 않고 Vault의 관리 기준만 현재 정상 상태에 다시 맞췄다.

## Before → After

```text
Before
MariaDB + Kubernetes Secret = PR #159 검증값
Ansible Vault = 다른 값
→ 관리 기준과 실제 소비 상태 불일치

After
Vault의 DB Credential 3종만 검증값으로 복구
→ Vault = MariaDB-01/02 = Kubernetes Secret
→ MaxScale / Replication 유지
→ 반복 실행 changed=0
```

## 관련 근거

- Infra Issue #169: https://github.com/seokpan/seokpan-infra/issues/169
- Infra PR #172: https://github.com/seokpan/seokpan-infra/pull/172
- PR #172 Merge Commit: `dc41791b90118fd1865d03b70cec7122dd31f95f`
- 선행 PR #159: https://github.com/seokpan/seokpan-infra/pull/159
