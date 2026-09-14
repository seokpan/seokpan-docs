[← 트러블슈팅 목차로 돌아가기](README.md)

# TS-038 — kube-apiserver AnonymousAuth·AlwaysAllow 조합 시 전체 401 Unauthorized

| 항목 | 내용 |
|---|---|
| **발생/발견 시기** | 2026-09-11 |
| **상태** | **해결** |
| **주 담당** | **김상희 — 데이터베이스·스토리지·복구** |
| **영향 범위** | DR-02 격리 검증 환경(단독 kube-apiserver, v1.36.2), 운영 API 서버 영향 없음 |

## 최초 문제

이슈 #156에서 격리 etcd 3-member를 backend로 하는 단독 kube-apiserver를
`--anonymous-auth=true`, `--authorization-mode=AlwaysAllow`로 기동한 뒤
`kubectl get namespace`를 실행하면 `Please enter Username:` 대화형
프롬프트가 뜨거나, `curl`로 직접 조회 시 다음과 같이 거부됐다.

```json
{"kind":"Status","status":"Failure","message":"Unauthorized","reason":"Unauthorized","code":401}
```

## 원인

kube-apiserver 기동 로그에 원인이 명시돼 있었다.

```
AnonymousAuth is not allowed with the AlwaysAllow authorizer.
Resetting AnonymousAuth to false. You should use a different authorizer
```

Kubernetes는 "모든 요청을 무조건 허용하는 authorizer(AlwaysAllow)"와
"인증 없이도 요청을 받아주는 anonymous 인증"을 동시에 쓰면 인증 절차
자체가 무의미해지므로, 기동 시점에 AnonymousAuth를 강제로 꺼버린다.
그 결과 이 구성에는 유효한 인증 수단이 하나도 남지 않아 모든 요청이
401로 거부됐다.

## 조치

`--anonymous-auth` 플래그를 제거하고, 이 격리 테스트 전용 self-signed
CA를 발급해 그 CA로 서명한 client certificate를 kubectl에 물리는
방식(`--client-ca-file`)으로 전환했다. `AlwaysAllow` authorizer는
그대로 유지했다 — 인증만 통과하면 권한 검사 없이 전부 허용되므로
검증 목적에는 이 조합으로 충분하다.

```bash
openssl req -x509 -new -nodes -key ca.key -subj "/CN=dr-restore-ca" -days 1 -out ca.crt
openssl req -new -key client.key -subj "/CN=dr-admin" -out client.csr
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 1 -out client.crt
```

kube-apiserver를 `--client-ca-file=ca.crt`로 재기동하고, kubectl에
`--client-certificate`/`--client-key`를 등록.

## 검증

- `kubectl get namespace` 정상 응답(스냅샷 시점 12개 네임스페이스 전부 조회)
- Namespace(12)/Deployment(21)/Secret(37) 전량 baseline과 `diff` 완전
  일치(종료 코드 0) 확인 — 인증 전환이 데이터 접근 결과에 영향 없음을
  같이 확인

## Before → After

```
Before
--anonymous-auth=true + --authorization-mode=AlwaysAllow
→ apiserver가 기동 시 AnonymousAuth를 강제로 false로 리셋
→ 유효 인증 수단 없음, 전체 401

After
self-signed CA 기반 client certificate 인증으로 전환
→ AlwaysAllow는 유지, 인증만 통과하면 전부 허용
→ kubectl 정상 응답
```

## 관련 근거

- Issue #156: https://github.com/seokpan/seokpan-infra/issues/156

## 후속 운영 기준

컨트롤 플레인 없이 kube-apiserver만 단독으로 띄우는 검증 작업(DR 테스트
등)에서는 anonymous 인증 대신 client certificate 인증을 기본값으로
삼을 것을 권장한다.
