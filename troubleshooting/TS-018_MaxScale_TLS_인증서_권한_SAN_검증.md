[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-018 — MaxScale TLS 적용 중 인증서 권한·상위 디렉터리·SAN 검증 문제

> 이 문서는 「石나가는 판단」 프로젝트에서 실제로 발생하거나 검증 과정에서 발견된 문제를 기록한 개별 트러블슈팅 보고서입니다. 링크를 열지 않아도 사건의 배경, 영향, 원인, 조치와 검증 결과를 이해할 수 있도록 작성합니다.

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-04 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | 정태훈(Backend DB 연동 경로), 최유준(`tls_deploy`/`internal_ca` role 소유), Backend 전체 |

## 문제 개요

MaxScale `Read-Write-Listener:3306`에 TLS를 적용하는 과정(이슈 #102)에서, 인증서 자체 발급은 정상 동작했음에도 실제 서비스 기동 단계에서 서로 다른 원인의 실패가 3단계에 걸쳐 연속으로 발생했다.

```text
2026-09-04 19:29:09   error  : (Read-Write-Listener); Bad path parameter
'/etc/pki/seokpan-ca/services/maxscale-01.crt': 13, Permission denied
```

## 발생 경위 및 원인 분석

### 1차: 인증서/키 파일 권한 불일치

`internal_ca` + `tls_deploy` role(최유준 작성)은 인증서와 키를 `owner: root, group: root`로 배포한다. 그런데 MaxScale은 systemd에서 `--user=maxscale`로 구동되므로, `root:root 0600`인 개인키를 MaxScale 프로세스가 읽을 수 없었다.

```text
maxscale.service 확인 결과: ExecStart=/usr/bin/maxscale --user=maxscale
```

### 2차: 상위 디렉터리 traverse 차단

파일 자체 권한(그룹 `maxscale`, `0640`)을 고친 뒤에도 동일한 Permission denied가 재현됐다. 원인은 인증서가 위치한 디렉터리의 **상위 디렉터리** `/etc/pki/seokpan-ca` 자체가 `drwxr-x--- root root`(0750)였던 것이었다. CentOS 하드닝 이미지의 기본 umask(077)로 인해 `tls_deploy`가 암묵적으로 생성한 중간 디렉터리가 이렇게 만들어졌고, `maxscale` 유저는 group에도 속하지 않아 통과 자체가 불가능했다.

### 3차: config-check가 root 실행을 거부

앞의 두 권한 문제를 해결한 뒤 배포 파이프라인에 추가한 `maxscale --config-check` 사전 검증 태스크가 다음 오류로 실패했다.

```text
alert  : MaxScale cannot be run as root.
```

MaxScale은 `--config-check` 같은 유틸리티 실행도 root로는 거부한다. `become_user: maxscale`로 실행 계정을 명시해 해결했다.

### 리뷰에서 추가로 발견: SAN 누락 및 재발급 미감지

리뷰어(tjung03)가 두 가지를 지적했다.

1. 인증서 SAN에 이슈 #99에서 이미 확정된 DB Canonical Endpoint(`db.seokpan.soldesk.store`)가 빠져 있었다. Backend는 MaxScale-01의 실제 IP가 아니라 Common VIP/Canonical Endpoint로 접속하므로, hostname 검증을 켜는 클라이언트에서 인증서 불일치가 발생할 수 있었다.
2. `tls_deploy` role의 재발급 판정이 만료 임박 여부(`checkend`)만 확인하고 SAN 변경은 감지하지 못해, host_vars의 SAN을 수정해도 자동으로 재발급되지 않았다.

1번은 SAN에 `DNS:db.seokpan.soldesk.store` 추가로, 2번은 이번 PR 범위에서는 기존 인증서 파일을 수동 삭제한 뒤 재실행하는 방식으로 우회했고, `tls_deploy` role 자체 수정은 소유자 담당 범위라 별도 이슈(#138)로 분리했다.

## 조치 요약

```text
1. maxscale role configure.yml에 권한 보정 태스크 3개 추가
   - 상위 디렉터리(/etc/pki/seokpan-ca) traverse 권한 0711
   - 인증서 디렉터리 group: maxscale, mode: 0750
   - 개인키 group: maxscale, mode: 0640
2. config-check 태스크에 become_user: maxscale 지정
3. host_vars의 SAN에 VIP(10.1.93.90) + Canonical Endpoint(db.seokpan.soldesk.store) 포함
4. 재발급 필요 시 기존 인증서 파일 수동 삭제 → tls_deploy 재실행 → (반드시) maxscale role
   재실행으로 권한 재보정 → 서비스 재기동, 순서를 Runbook으로 확정
```

## 검증

- `--skip-ssl` 없이 VIP(10.1.93.90) 접속 성공, `Cipher in use is TLS_AES_256_GCM_SHA384, cert is OK`
- `db.seokpan.soldesk.store` + `--ssl-ca` + `--ssl-verify-server-cert`로 mariadb-01/02 양쪽에서 hostname/CA 검증까지 통과 확인
- 인증서 SAN에 `DNS:maxscale-01.seokpan.internal, DNS:db.seokpan.soldesk.store, IP:192.168.53.40, IP:10.1.93.90` 전부 포함 확인

## 부록 — MaxScale `--check --diff` 비밀번호 평문 노출 (PR #104)

같은 계정/설정 코드화 흐름에서 발견된 별개 문제다. `maxscale.cnf` 배포 태스크에 `diff: false`가 없어, `--check --diff` 실행 시 `maxscale_monitor_password`/`repl_user_password`가 콘솔·CI 로그에 평문으로 출력됐다. `diff: false` 한 줄 추가로 해결했으며, 파일 내용 자체는 바뀌지 않으므로 서비스 재시작은 필요 없었다. `mariadb` role의 `custom.cnf.j2`는 비밀번호를 포함하지 않는 파일이라 동일 조치가 불필요함도 함께 확인했다.

## Before → After

```text
Before
tls_deploy가 배포한 인증서/키를 root:root 그대로 사용
→ MaxScale(non-root 유저)이 못 읽음
→ 상위 디렉터리까지 막혀 있어 파일 권한만으론 해결 안 됨
→ config-check도 root 실행 거부
→ SAN에 실제 접속 주소(VIP/Canonical Endpoint) 누락

After
소비 서비스(maxscale) role에서 디렉터리·키 권한 개별 보정
→ config-check는 become_user: maxscale로 실행
→ SAN에 VIP + Canonical Endpoint 포함
→ 재발급 시 권한 재보정이 필요하다는 것을 Runbook으로 명시
```

## 관련 근거

- Issue #102: https://github.com/seokpan/seokpan-infra/issues/102
- PR #137: https://github.com/seokpan/seokpan-infra/pull/137
- Issue #138 (tls_deploy SAN 미감지, 후속): https://github.com/seokpan/seokpan-infra/issues/138
- PR #104 (비밀번호 노출 방지): https://github.com/seokpan/seokpan-infra/pull/104
