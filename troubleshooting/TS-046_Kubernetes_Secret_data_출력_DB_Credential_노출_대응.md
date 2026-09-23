[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-046 — Kubernetes Secret 검증 중 DB Credential을 복원할 수 있는 값이 출력된 문제

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-08 |
| **상태** | **해결 / 노출 가능 Credential 회전 완료** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | Backend DB Runtime·Migration Secret 공급 자동화, `identity_svc`·`game_svc`·`db_admin` Credential |

## 최초 문제

Backend DB Runtime·Migration Secret 공급 자동화를 검증하는 과정에서 Kubernetes Secret 구조를 수동으로 확인하면서 Secret의 base64 `data`까지 출력하는 방식을 사용했다.

Kubernetes Secret의 `data`는 암호화된 값이 아니라 base64로 인코딩된 값이므로, 출력된 내용에서 기존 DB Connection Credential을 복원할 수 있었다.

따라서 해당 출력에 포함된 다음 세 계정의 Credential은 더 이상 안전한 값으로 간주할 수 없었다.

```text
identity_svc
game_svc
db_admin
```

## 원인

문제의 원인은 Kubernetes Secret의 존재·Type·Key 구조만 확인하면 되는 검증에서 실제 `data` 값까지 출력한 것이다.

```text
필요한 검증
Secret 존재
+ Namespace
+ Type
+ Key 이름

실제 초기 검증
Secret data까지 출력
→ base64 값 노출
→ DB Credential 복원 가능
```

base64 인코딩을 민감정보 보호 수단으로 취급할 수 없으므로, 실제 값이 Git에 저장되지 않았더라도 터미널 출력에 노출된 Credential은 회전이 필요하다고 판단했다.

## 조치

기존 검증 방식은 즉시 폐기했다.

Credential 회전은 기존 계정·권한 구조를 바꾸지 않고 다음 순서로 수행했다.

```text
신규 Credential 생성
→ 기존 Ansible Vault 변수 값 갱신
→ MaxScale에서 현재 MariaDB Master 확인
→ 실제 Master의 세 계정 Password 회전
→ Replication 상태 확인
→ 회전된 Vault 값으로 Kubernetes Secret 재공급
→ Secret 실제 값 없이 구조만 검증
```

기존 `mariadb_account` Role은 계정 재실행 시 Password를 자동 변경하지 않는 `update_password: on_create` 정책을 유지했다.

이번 회전은 노출 대응을 위한 일회성 작업으로 별도 임시 Playbook에서 `update_password: always`를 사용했고, 실행 후 지속 Desired State에는 포함하지 않았다.

최종 자동화에서는 다음 기준을 적용했다.

- 실제 Password와 완성된 DB URL을 출력하지 않음
- 민감 Task는 `no_log: true` 사용
- 실제 DB URL 구조 검증 시 Credential-bearing 값을 `argv`로 전달하지 않고 `stdin` 사용
- URL 인코딩 검증은 비민감 synthetic Credential로 수행
- Kubernetes Secret은 값이 아니라 Resource 존재·Type·Key·관리 Label만 확인

## 검증

Credential 회전 후 MariaDB와 Kubernetes Secret을 다시 확인했다.

MaxScale 기준:

```text
mariadb-01   Master, Running
mariadb-02   Slave, Running
GTID         일치
```

회전된 Vault 값으로 두 Secret을 다시 공급했다.

```text
backend-db-runtime
- SEOKPAN_IDENTITY_DATABASE_URL
- SEOKPAN_GAME_DATABASE_URL

backend-db-migration
- SEOKPAN_MIGRATION_DATABASE_URL
```

재공급 시 실제 Secret 내용이 변경돼 `changed=2`가 발생했고, 동일한 상태에서 다시 실행했을 때:

```text
changed=0
failed=0
```

으로 수렴했다.

최종 검증 Output에는 실제 Credential과 완성된 DB URL이 포함되지 않았고, Runtime Secret과 Migration Secret의 Key 경계도 유지됨을 확인했다.

## Before → After

```text
Before
Secret 구조 확인 과정에서 base64 data까지 출력
→ DB Credential 복원 가능
→ 노출된 Credential을 안전한 값으로 볼 수 없음

After
노출 가능 Credential 3종 회전
→ Vault / MariaDB / Kubernetes Secret 재공급
→ Secret 값 비출력 검증으로 변경
→ 동일 상태 재실행 changed=0
```

## 관련 근거

- Infra Issue #150: https://github.com/seokpan/seokpan-infra/issues/150
- Infra PR #159: https://github.com/seokpan/seokpan-infra/pull/159
- PR #159 Merge Commit: `0d219b107e40577a993637c6af638e01183562f9`
