# 石나가는 판단 MVP 구현 기준

이 문서는 01~08 공식 기획·설계와 [프로젝트 변경·결정 이력](PROJECT_CHANGES.md)을 실제 구현에서 일관되게 적용하기 위한 공용 기준이다.
원 기획·설계를 대체하지 않으며, 07에서 정한 MVP 범위를 구현 편의를 이유로 확대하거나 축소하지 않는다.

## 1. 적용 기준과 상태

적용 우선순위는 다음과 같다.

1. 07의 MVP 포함·제외 범위
2. 01의 포함 기능 상세 동작
3. 03~06의 역할·기술·물리·검증 기준
4. 사용자가 승인하고 `PROJECT_CHANGES.md`에 기록한 PDF 이후 변경
5. 이 문서의 MVP 구현 상세

주요 근거는 D07 pp.2~8·12~14, D01 pp.3~5·13~53, D03 pp.2~8, D04 pp.3~18,
D05 pp.2~14, D06 pp.3~23이다. 페이지는 PDF의 1부터 시작하는 순서다.

실제 코드와 작업 상태는 각 구현 Repository가 기준이다. 구현이 위 요구사항과 다르면 구현 측 수정 대상으로 다루며,
구현 상태만을 근거로 원 요구사항을 변경하지 않는다.

| 구분 | 현재 상태 |
| --- | --- |
| MVP·구현 방향 | 확정 |
| Backend·Frontend Windows Compatibility Spike | 통과 |
| Linux Container 동일 Lock·Image 실행 | 검증 대기 |
| 실제 MariaDB·Redis Provider 통합 | 검증 대기 |
| Harbor·Gateway·GitOps·Argo CD 통합 | 검증 대기 |
| Application Scaffold·기능 구현 | 미착수 |

결정, 정적 자산 존재, 담당자 실행 보고, 직접 Runtime 검증을 같은 완료 상태로 표시하지 않는다.

## 2. Repository 책임

| Repository | 책임 |
| --- | --- |
| `seokpan/seokpan-docs` | 공용 기획·설계, 승인된 변경, 여러 구현 저장소가 소비할 기준 |
| `seokpan/seokpan-app` | Frontend·Backend Source, Test, DB Model·Migration, Redis Key/Lua 계약, Containerfile, Jenkinsfile |
| `seokpan/seokpan-infra` | VM·OS·Network, Ansible, Kubernetes Bootstrap/Add-on, MariaDB·MaxScale, Harbor·CA 신뢰 기반 |
| `seokpan/seokpan-gitops` | Namespace/RBAC, Application·Platform Workload, Config·Secret 참조, Gateway Route, Argo CD Desired State |

Application Source와 Kubernetes Desired State를 한 Repository에 섞지 않는다.
Jenkins는 Application을 직접 `kubectl apply`하지 않고 검증된 Image와 GitOps 변경 흐름을 사용한다.

## 3. 점진적 구현 순서

```text
기존 Schema·MVP 계약 확인
  → Pure Domain 규칙과 Test
  → 기존 Schema를 잇는 Migration Baseline
  → Port·Fake·Adapter Contract
  → 실제 MariaDB·Redis Contract Test
  → Headless HTTP/WebSocket First Success
  → Frontend First Success
  → Container·Harbor·Gateway·GitOps 통합
  → 장애·복구·재접속·멱등성·대표 부하 검증
```

UI나 실제 Provider가 준비되지 않아도 Domain·Fake·Contract Test로 진행할 수 있다.
다만 Fake 성공은 MariaDB·Redis·Kubernetes 통합 완료를 의미하지 않는다.

## 4. Backend 기준

- 하나의 FastAPI Application과 하나의 Backend 배포 단위를 사용하는 Modular Monolith로 구현한다.
- 내부는 AUTH, LOBBY/ROOM, GAME, VOTE, REALTIME 역할을 기능 단위 Module로 나눈다.
- Domain은 FastAPI, SQLAlchemy, Redis Client, HTTP 상태 코드와 Kubernetes 설정을 직접 참조하지 않는다.
- 외부 I/O는 Application Port와 Adapter를 통해 연결하며, 역할별 Microservice로 분할하지 않는다.

| 영역 | 기준 |
| --- | --- |
| Runtime | 표준 GIL CPython 3.13.15 |
| Project·Lock | uv 0.12.5, `pyproject.toml`, `uv.lock`, `.python-version` |
| HTTP/WebSocket | FastAPI 0.141.1, Uvicorn 0.52.4, Pydantic Settings |
| DB | SQLAlchemy 2.0.52 AsyncIO, Alembic 1.19.1, asyncmy 0.2.14 |
| Redis | redis-py 8.1.0의 `redis.asyncio`, Version 관리 Lua Script |
| 품질 Gate | pytest 계열, Ruff, mypy strict |

Ansible Controller의 Project Python 3.12.13과 Application Backend Python 3.13.15는 목적과 실행환경이 다르므로 독립 유지한다.
Application Container는 Kubernetes Node나 Ansible Controller의 System Python에 의존하지 않는다.

## 5. HTTP·WebSocket 계약 방향

- 상태 조회와 변경 명령은 `/api/v1` HTTP JSON API를 기본으로 한다.
- WebSocket `/ws/v1`은 서버 권위 Snapshot과 상태 변경 Event 전달에 사용한다.
- HTTP 오류는 RFC 9457 `application/problem+json`과 안정적인 Domain Error Code를 사용한다.
- WebSocket 메시지는 Version이 있는 JSON Envelope를 사용하고 `event_id`, `room_id`, `game_id`, `state_version`을 필요한 범위에서 전달한다.
- 연결과 재연결 시 Snapshot으로 수렴한다. Event와 Redis Pub/Sub을 무제한 Replay 원본으로 사용하지 않는다.
- Redis 서버측 Session Cookie를 HTTP와 WebSocket Upgrade에서 함께 사용하며 Token을 URL Query에 넣지 않는다.

Endpoint·Event 전체 목록, Cookie TTL·CSRF Header, 오류 코드 상세는 Scaffold 전에 계약 v0로 확정한다.

## 6. MariaDB·Redis 기준

| 데이터 | 권위 저장소 |
| --- | --- |
| Member, MemberStats, Game, Move, GameResult, RatingHistory | MariaDB |
| Session, Room, Participant, Ready, Game/Turn Runtime, 현재 Vote, 재접속 Snapshot | Redis |

- DB 담당자의 실제 Runtime 조회 결과와 사용자가 제공한 `/root/stone_game_schema_v1.sql`·DDL 출력을 대조해 일치가 보고된
  `member`, `member_stats`, `game`, `game_participant`, `move`, `game_result`, `rating_history` 7개 Table을 초기 Schema 기준으로 재사용한다.
  이는 이 문서 작업에서 DB에 직접 접속해 다시 실행 검증했다는 뜻이 아니다.
- `seokpan-app`이 SQLAlchemy Model과 Alembic Revision을 소유한다. 기존 DB는 DDL·Checksum 확인 후 승인된 Revision으로 채택하며 초기 Create를 재실행하지 않는다.
- Migration은 Backend Replica 시작마다 실행하지 않고 승인된 단일 선행 Job 또는 운영 절차로 실행한다.
- Backend는 시점에 따라 바뀔 수 있는 MariaDB Master IP를 고정하지 않고 MaxScale/Common Endpoint를 사용한다.
- Redis 투표·마감은 Lua로 원자 처리하고, 공식 Move·Result·Rating 중복 방지는 MariaDB Transaction과 Constraint가 담당한다.
- MariaDB Commit 후 Redis를 갱신하며, Redis 갱신 실패 시 MariaDB 확정 결과로 멱등 재동기화한다.
- 식별자 기본 형식은 소문자 하이픈 UUIDv4이며 `game.room_id VARCHAR(64)`는 호환성을 유지한 채 신규 값에 UUIDv4를 사용한다.
- Redis Key Prefix는 `stone:v1:`이며 상세 Key·TTL은 계약 v0에서 Freeze한다.

## 7. Frontend 기준

| 영역 | 기준 |
| --- | --- |
| UI | React 19.2.8, React DOM 19.2.8 |
| 언어 | TypeScript 5.9.3 strict |
| Build | Vite 8.2.2, `@vitejs/plugin-react` 6.1.1 |
| 개발 Runtime | Node.js 24 LTS, npm 12와 `package-lock.json` |
| Routing | React Router 7.18.3 |
| HTTP Type | `openapi-typescript` 7.13.0 생성 Type과 작은 `fetch` Wrapper |
| 상태 | 기능 단위 Context/`useReducer`, 서버 Snapshot과 로컬 UI 상태 분리 |
| Styling | CSS Modules와 공통 CSS Token |

- Browser는 MariaDB·Redis에 직접 접근하거나 서버 권위 상태를 소유하지 않는다.
- `state_version`이 연속이면 Event를 적용하고, 누락되거나 재연결되면 Snapshot을 다시 받아 수렴한다.
- Move·Vote 마감·승패·Rating을 서버 확인 전에 낙관적으로 확정 표시하지 않는다.
- Frontend는 Nginx 정적 Application으로 제공하며 Browser History Route는 SPA Fallback을 사용한다.
- ANALYSIS·채팅·고급 UI를 First Success의 선행조건으로 만들지 않는다.
- Frontend 화면 골격과 시각 방향을 구체화하기 전에 사용자에게 참고 UI Mockup을 요청한다.
- 제공된 Mockup은 공식 요구사항이나 Pixel-perfect 명세가 아닌 방향성 참고자료다. docs MVP·확정 계약·접근성·실제 사용자 흐름을 우선하며, Mockup의 자체 오류나 범위 밖 기능을 그대로 구현하지 않는다.

## 8. Image·실행·환경 기준

| Workload | Runtime | Port | 최초 Replica 기준 |
| --- | --- | --- | --- |
| Backend | `python:3.13.15-slim-trixie`, Uvicorn 단일 Process | 8000 | 2 |
| Frontend | `nginxinc/nginx-unprivileged:1.30.4-alpine3.24` | 8080 | 2 |

- Backend와 Frontend는 별도 Image·Workload로 배포한다.
- 외부 주소는 `https://game.seokpan.soldesk.store`, Registry는 `harbor.seokpan.soldesk.store`다.
- Harbor 관련 명칭은 VM `harbor-01`, Ansible Inventory Host `harbor`, 서비스 FQDN `harbor.seokpan.soldesk.store`, IP `192.168.53.61`로 구분한다. 현행 Ansible 자산은 OS hostname 자체를 설정하지 않는다.
- Image Tag는 `git-<12자리-main-commit>` 형식을 사용하고 동일 Tag를 덮어쓰지 않는다. GitOps는 검증된 Digest를 소비한다.
- Backend와 Frontend Kubernetes Service Port 이름은 `http`를 사용한다. 현재 GitOps의 Application ServiceMonitor에 남은 `metrics` Port 가정은 앱 통합 때 수정한다.
- Backend Health는 `/health/startup`, `/health/live`, `/health/ready`, Metric은 `/metrics`를 사용한다. DB·Redis 장애를 Liveness 실패로 처리하지 않는다.
- Frontend Health는 `/health/live`를 사용한다.
- 설정 Prefix는 `SEOKPAN_`이다. 공용 이름은 `SEOKPAN_ENVIRONMENT`, `SEOKPAN_LOG_LEVEL`, `SEOKPAN_PUBLIC_BASE_URL`,
  `SEOKPAN_ALLOWED_ORIGINS`, `SEOKPAN_TRUSTED_HOSTS`, `SEOKPAN_DATABASE_URL`, `SEOKPAN_REDIS_URL`, `SEOKPAN_INSTANCE_ID`를 사용한다.
  DB·Redis 연결 정보는 Kubernetes Secret 참조로 주입하며 실제 Secret 값과 CA Private Key를 Git·Image·Log에 기록하지 않는다.
- 두 Workload는 Non-root, Capability Drop, `RuntimeDefault` seccomp와 읽기 전용 Root Filesystem을 기본으로 한다.
- Backend는 Uvicorn 단일 Process로 실행하고 Migration·Seed를 시작 명령에 숨기지 않는다. 종료 유예는 최초 45초 기준이며 WebSocket은 재연결 후 Snapshot으로 수렴한다.
- Resource Request/Limit은 실행 측정 전 임의의 수치로 확정하지 않는다.

## 9. CI·통합·Evidence Gate

| Gate | 목적 |
| --- | --- |
| P0 Local | Lock, Format/Lint, Type Check, 빠른 Unit·Component Test |
| P1 PR | P0, Backend/Frontend Build·Test, 계약·Migration 정적 검증 |
| P2 Main Image | Linux 동일 Lock, Image Build·Health, SBOM·Provenance·Scan, Harbor Push·Digest |
| P3 Provider Integration | MariaDB·Redis, Migration, 2 Backend Replica, HTTPS/WSS, Session·재접속, Metric |
| P4 MVP Acceptance | Headless·Browser First Success, Race·멱등·장애·복구·대표 부하 Evidence |

주 CI는 `seokpan-app/Jenkinsfile`의 Declarative Pipeline이다.
PR에서는 Harbor·GitOps·실제 Provider Credential을 사용하지 않고, `seokpan-app`의 보호된 main에서 검증된 Image만 Harbor와 GitOps 후보로 진행한다.
Image Scan은 CRITICAL과 수정 가능한 HIGH를 차단하며 예외는 CVE·영향·대안·만료일·승인자를 기록한다.

MVP 완료에는 정상 흐름뿐 아니라 투표 경합, stale 요청, 중복 Move/Result 방지, DB Commit 후 Redis 재동기화,
WebSocket 재접속과 Snapshot 수렴, 장애로 인한 오패배·Rating 감소 방지 검증이 포함된다.

## 10. Windows 개발과 Linux 실행 호환 기준

- Windows Host PC에서 Source와 문서를 작성하되 실제 Application 실행 기준은 CentOS Stream 9·Linux Container다.
- `seokpan-app`의 Source, Shell, YAML, Containerfile/Dockerfile, Jenkinsfile과 Nginx 설정은 UTF-8과 LF를 사용한다.
- Windows 전용 Batch 파일이 필요한 경우에만 CRLF를 허용한다.
- `seokpan-app`은 `.gitattributes`로 Git 개행 정책을, `.editorconfig`로 Editor의 UTF-8·개행·마지막 줄 기준을 명시한다.
- Windows 절대경로와 대소문자를 무시한 Import·파일 참조를 코드에 넣지 않는다.
- Windows에서 설정하기 어려운 실행 비트는 Git Index에 기록하고 Linux Container Gate에서 확인한다.
- Windows Test 성공은 Linux 실행 성공을 대신하지 않으며, 동일 Lock의 Linux Build·Test·Entrypoint·Non-root 실행을 P2 전에 검증한다.
- Markdown 문서는 실행 자산과 구분하며 특정 Worktree EOL을 강제하지 않는다. Git 정규화와 파일 내 혼합 개행 방지만 적용한다.

## 11. 아직 Freeze하지 않은 항목

- HTTP Endpoint·WebSocket Event·Domain Error Code 전체 목록
- Cookie TTL·갱신·CSRF Header·Password Hash 상세
- Redis Key·TTL·Resolver Lease와 Lifecycle 상세
- 기존 Schema 보완 Migration과 실제 적용 Owner·시점
- DB·Redis Endpoint 및 Kubernetes Secret Resource 이름
- GitOps Application 경로와 Migration Sync 순서
- 측정 기반 Resource Request/Limit과 Evidence Retention 기간

위 항목은 관련 구현 전에 앞 결정과 Provider 인계를 입력으로 계약 v0에 기록한다.
뒤 결정이 앞 기준을 변경해야 하면 기존 내용을 조용히 덮지 않고 영향·Migration·재검증 Gate를 기록한다.

## 12. 변경 관리

공용 계약 변경은 관련 App·Infra·GitOps 영향을 대조하고 사용자 검토 후 `PROJECT_CHANGES.md`와 이 문서를 함께 갱신한다.
Issue·Branch·Commit·Push·PR·Review·Merge 등 GitHub 변경은 저장소 소유권을 확인하고 사전 승인을 받은 뒤 수행한다.
