[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-046 — Kubernetes Secret 확인 중 DB 비밀번호를 복원할 수 있는 값이 출력된 문제

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-08 |
| **상태** | **해결 / 관련 DB 비밀번호 변경 완료** |
| **주 담당** | **정태훈 — Kubernetes 플랫폼 및 애플리케이션 통합** |
| **영향 범위** | Backend·Migration DB Secret 자동화, `identity_svc`·`game_svc`·`db_admin` 비밀번호 |

## 최초 문제

Backend와 Migration에서 사용할 DB Secret을 Ansible로 만드는 작업을 검증하던 중, Kubernetes Secret의 구조를 확인하기 위해 `data` 값까지 터미널에 출력했다.

Kubernetes Secret의 `data`는 암호화된 값이 아니라 base64로 인코딩된 값이다. 따라서 출력된 값을 디코딩하면 DB 접속에 사용하는 비밀번호를 다시 얻을 수 있었다.

해당 출력에 다음 세 계정의 비밀번호가 포함될 수 있었기 때문에 기존 값을 더 이상 안전한 값으로 사용하지 않기로 했다.

```text
identity_svc
game_svc
db_admin
```

## 원인

Secret 검증에 필요한 정보보다 더 많은 값을 출력한 것이 원인이었다.

필요했던 확인 항목은 다음과 같았다.

```text
Secret 존재 여부
Namespace
Type
Key 이름
```

하지만 초기 검증에서는 실제 `data` 값까지 출력했다.

```text
Secret data 출력
→ base64 값 확인 가능
→ 디코딩하면 DB 비밀번호 복원 가능
```

base64는 값을 다른 형태로 표현할 뿐 비밀번호를 보호하는 암호화 방식이 아니다.

## 조치

기존 검증 방법은 바로 중단했다.

노출 가능성이 생긴 세 계정의 비밀번호는 다음 순서로 새 값으로 변경했다.

```text
새 비밀번호 생성
→ Ansible Vault의 기존 변수 값 변경
→ MaxScale에서 현재 MariaDB Master 확인
→ Master에서 세 계정 비밀번호 변경
→ MariaDB 복제 상태 확인
→ 변경한 비밀번호로 Kubernetes Secret 다시 생성
→ Secret 실제 값은 출력하지 않고 구조만 확인
```

기존 `mariadb_account` Role의 `update_password: on_create` 설정은 그대로 유지했다. 일반적인 Ansible 재실행 때 기존 계정 비밀번호가 자동으로 바뀌지 않도록 하기 위해서다.

이번 비밀번호 변경만 별도의 임시 Playbook에서 `update_password: always`로 실행했고, 작업이 끝난 뒤 해당 Playbook은 계속 사용하는 자동화에 포함하지 않았다.

이후 검증 방식도 다음과 같이 바꿨다.

- 실제 비밀번호와 완성된 DB URL을 출력하지 않음
- 비밀번호를 다루는 Task에 `no_log: true` 적용
- 실제 DB URL을 검사할 때 명령행 인자로 넘기지 않고 `stdin`으로 전달
- URL 인코딩 시험은 실제 비밀번호 대신 임의의 테스트 값으로 수행
- Kubernetes Secret은 실제 값이 아니라 존재 여부·Type·Key 이름·관리 Label만 확인

## 검증

비밀번호를 변경한 뒤 MariaDB 상태를 확인했다.

```text
mariadb-01   Master, Running
mariadb-02   Slave, Running
GTID         일치
```

변경한 비밀번호로 다음 Secret을 다시 만들었다.

```text
backend-db-runtime
- SEOKPAN_IDENTITY_DATABASE_URL
- SEOKPAN_GAME_DATABASE_URL

backend-db-migration
- SEOKPAN_MIGRATION_DATABASE_URL
```

Secret 내용이 바뀐 실행에서는 다음 결과를 확인했다.

```text
changed=2
failed=0
```

같은 설정으로 다시 실행했을 때는 다음과 같았다.

```text
changed=0
failed=0
```

최종 확인 과정에서는 실제 비밀번호와 완성된 DB URL이 출력되지 않았고, Backend용 DB 정보와 Migration용 DB 정보가 서로 섞이지 않은 것도 확인했다.

## Before → After

```text
Before
Secret 구조를 확인하면서 data 값까지 출력
→ base64 값을 디코딩하면 DB 비밀번호 복원 가능

After
관련 DB 비밀번호 3개를 새 값으로 변경
→ Vault / MariaDB / Kubernetes Secret에 같은 값 반영
→ 실제 Secret 값은 출력하지 않는 방식으로 검증 변경
→ 같은 설정으로 다시 실행했을 때 changed=0
```

## 관련 근거

- Infra Issue #150: https://github.com/seokpan/seokpan-infra/issues/150
- Infra PR #159: https://github.com/seokpan/seokpan-infra/pull/159
- PR #159 Merge Commit: `0d219b107e40577a993637c6af638e01183562f9`
