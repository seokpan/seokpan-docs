# 石나가는 판단 - 확장 호환형 MVP 도출

원 목표 구조의 역할·Endpoint·데이터 경계를 유지하면서 4인 팀이 우선 구현·검증할 최소 실행 구조를 정의한다.

> **문서 기준** Physical Server 4대에서 Windows 10 Enterprise, VMware Workstation Pro 25H2, CentOS Stream 9 Guest 기반을 유지한다. 원 목표 18개 VM에서 lb-02와 maxscale-02를 제외한 16개 VM을 MVP로 사용하고, ANALYSIS 런타임은 원 목표 확장으로 둔다. UI 목업은 시각 참고자료일 뿐 기능·권한·상태·복구 계약의 판단 근거가 아니다.

이 문서는 서비스 요구사항, 핵심 문제와 검증 목표, 논리 역할, 기술 선택, 물리 아키텍처, Ansible 자동화·테스트 설계를 실제 일정 안에서 구현 가능한 범위로 변환한다. 축소의 목적은 기술 수를 줄이는 것이 아니라 중복 인스턴스와 추가 HA 복잡도를 늦추어 First Success와 정량 검증 시간을 확보하는 데 있다.

## 목차

> 1\. 문서 목적·범위·MVP 정의
>
> 2\. 선행 설계에서 보존할 가치와 판단 원칙
>
> 3\. Must·Should·Could·확장 범위
>
> 4\. 원 목표 구조와 MVP 구조 비교
>
> 5\. MVP 물리·네트워크·Endpoint 구조
>
> 6\. MVP 서비스 기능·상태 계약
>
> 7\. 구성요소별 유지·단일화·제외 판단
>
> 8\. 구현 순서와 First Success Milestone
>
> 9\. 핵심 검증과 Evidence 설계
>
> 10\. WBS·선후관계·Critical Path
>
> 11\. 4인 역할·Provider·Consumer
>
> 12\. 실행 일정·동결·Go/No-Go
>
> 13\. MVP 완료 Gate
>
> 14\. 원 목표 구조 확장 경로
>
> 15\. 리스크·제약·주장 경계
>
> 16\. 최종 기획안·실시설계 인계 및 완료 판정

## 1. 문서 목적·범위·MVP 정의

MVP는 원 목표 구조를 대체하는 별도 아키텍처가 아니다. 최종 역할·인터페이스·데이터 책임·핵심 기술 선택과 검증축을 유지하면서 최초 서비스 성립에 필요하지 않은 중복 인스턴스, 추가 HA, 고급 운영 기능을 뒤로 미룬 최소 실행 부분집합이다.

| **구분**      | **정의**                                                                    | **이 문서의 판정**                                                                                        |
|---------------|-----------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| 프로젝트 전체 | 서비스 기획부터 구현·검증·발표·포트폴리오까지의 전체 활동                   | MVP와 원 목표 구조를 모두 포함한다.                                                                       |
| 원 목표 구조  | 물리 아키텍처와 자동화·테스트 설계에서 정의한 4대 Physical Server·18VM 구조 | 18VM에는 lb-02·maxscale-02를 포함하며, Kubernetes 내부에는 ANALYSIS Workload와 관련 검증 범위를 포함한다. |
| MVP 구조      | 원 목표 역할과 Endpoint를 유지한 16VM 우선 구현 구조                        | First Success와 핵심 검증 완료 전까지 구현 기준으로 사용한다.                                             |
| 원 목표 확장  | MVP 완료 후 추가 인스턴스·Workload·검증을 더하는 단계                       | 9월 15일 Go/No-Go 통과 시에만 수행한다.                                                                   |

> **핵심 결정** MVP 성공은 화면 시연만으로 판정하지 않는다. 서비스 기능, 상태·데이터, 인프라, 자동화, 검증·Evidence의 다섯 Gate가 모두 닫혀야 한다.

## 2. 선행 설계에서 보존할 가치와 판단 원칙

| **보존 대상**                   | **MVP에서 유지하는 이유**                                              | **축소 시 허용 범위**                                                       |
|---------------------------------|------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| 실시간 집단 투표 게임           | 실시간 연결·동시성·상태 일관성을 검증하는 실제 Workload다.             | 부가 UI는 줄일 수 있으나 투표·Move·Result 권위는 줄이지 않는다.             |
| On-premise + Ansible            | 프로젝트의 핵심 구현·포트폴리오 가치다.                                | Windows·VMware·VM 생성은 수동 기반으로 두고 CentOS Guest 이후를 자동화한다. |
| Kubernetes HA·Scale-out         | CP quorum, Worker 장애, Replica·HPA를 검증한다.                        | CP 3대·Worker 2대는 유지한다.                                               |
| Runtime State / Persistent Data | Redis 상태와 MariaDB 영속 데이터의 책임·복구 경계가 핵심이다.          | Redis HA는 줄여도 AOF/PVC·복구는 유지한다.                                  |
| Stable Endpoint                 | MVP 이후 인스턴스를 추가해도 Application 설정을 다시 바꾸지 않게 한다. | 단일 LB·MaxScale이어도 동일 주소와 Port를 사용한다.                         |
| CI/CD·GitOps·관측성             | 구축·배포·장애 결과를 자동화하고 증거로 연결한다.                      | 도구 자체 HA와 장기 보존은 뒤로 미룬다.                                     |
| Backup/Restore·DR               | 복제와 별개로 영속 데이터 복구 가능성을 증명한다.                      | 완전 자동 DR 대신 승인된 복구와 RTO/RPO 측정을 유지한다.                    |

### 2.1 판단 순서

- 고급 운영 기능을 먼저 제외하고, 자동화 고도화와 동일 역할의 두 번째 인스턴스를 다음으로 줄인다.

- 보조 기능은 핵심 권위 경로와 분리될 때 후속으로 이동한다.

- 핵심 기술 자체 제거는 마지막 수단으로만 검토한다.

- Application Endpoint 변경, 데이터 Migration, Ansible Role 전면 재작성으로 이어지는 임시 구조는 채택하지 않는다.

- 구성요소 버전은 자동화·테스트 설계의 안정 Release 고정값을 유지하고 RC·alpha·beta·latest를 사용하지 않는다.

- 구현·측정하지 않은 성능·복구·HA 결과를 완료된 성과처럼 작성하지 않는다.

## 3. Must·Should·Could·확장 범위

| **분류**     | **영역**       | **MVP 범위**                                                                                                                                                | **미구현 시 판정**                                           |
|--------------|----------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Must         | 사용자·입력    | Guest 임시 진입, Member 가입·로그인·로그아웃, Member만 방 생성, 회원가입·방 생성 필수 입력의 서버 검증                                                      | 권한 우회나 유효하지 않은 입력이 핵심 상태를 변경하면 미완료 |
| Must         | 로비 기본      | 방 목록, 공개 여부·상태·인원 표시, 기존 방 입장                                                                                                             | 사용자가 정상적으로 게임 방을 찾고 입장할 수 없으면 미완료   |
| Must         | 방·대기방      | 공개·비공개·비밀번호, 팀 선택, Ready, 최소 Ready·최대 인원, 투표 시간, 방장 승계, 시작 조건 검증                                                            | 게임 시작 전 권한·상태 계약이 깨지면 미완료                  |
| Must         | 게임           | 15×15 렌주, 팀별 교대 투표, 실시간 집계, 동률 무작위 선택, 단일 착수, Pass·무투표·이탈 처리                                                                 | Vote·Move·Result 서버 권위가 성립하지 않으면 미완료          |
| Must         | 종료·기록      | 승·패·무·몰수·공동 패배·시스템 무효, 결과 화면, 대기방 복귀·나가기, Member 전적·Elo Rating·확정 Move·결과 멱등 처리                                         | 결과 또는 영속 기록을 신뢰할 수 없으면 미완료                |
| Must         | 실시간·복구    | 30초 재접속 유예, Room·Team·Game 복원, 단일 Backend 장애 후 활성 상태 복구, 시스템 장애 오패배 방지                                                         | 활성 게임 복구와 장애·사용자 행위 구분이 실패하면 미완료     |
| Must         | 인프라·자동화  | 16VM, Network·Kubernetes·MariaDB·MaxScale·Redis·NFS·CI/CD·관측성·Backup/Restore·Ansible                                                                     | 핵심 실행 경로나 검증축이 성립하지 않으면 미완료             |
| Must         | 관측성         | Prometheus·Grafana·Loki·Alloy·Alertmanager, 프로젝트용 Grafana 운영 대시보드, E-mail 알림                                                                   | 장애 Timeline과 핵심 Evidence를 재구성할 수 없으면 미완료    |
| Should       | 서비스 보조    | 실시간 접속자, 로비·방 채팅(구현 시 채팅 입력은 서버에서 길이·형식·허용 범위를 검증), 방장 강퇴, 진행 중 방 관전, 랭킹 화면, 게임 방법, 사용자 메뉴, 재경기 | 핵심 게임은 성립하므로 Functional Freeze를 막지 않음         |
| Could        | 표현·운영 보조 | 목업 수준의 세부 UI 연출, 단순 정적 운영 링크 포털, 선택적 Discord 알림                                                                                     | 일정 여유가 있을 때만 수행                                   |
| 원 목표 확장 | 고가용성·분석  | lb-02·maxscale-02·Keepalived/VRRP, ANALYSIS 런타임, 관련 HA·부하·Failure Domain 검증                                                                        | MVP 완료 후 Go/No-Go 대상                                    |
| 1차 비대상   | 고급 운영      | 별도 관리자 백엔드·관리자 웹, Redis Sentinel/Cluster, 완전 자동 DR, WAF                                                                                     | 현재 프로젝트 성과로 주장하지 않음                           |
| 2차 프로젝트 | Hybrid·Cloud   | OpenShift·ROSA·Terraform·Cloud 자동 확장                                                                                                                    | 1차 성공을 방해하지 않도록 이관                              |

로비 채팅과 방 채팅은 서로 다른 전달 범위로 분리하며 메시지를 교차 전달하지 않는다. 진행 중 입장한 관전자는 입장한 방의 채팅만 수신한다.

### 3.1 UI 목업의 위치

UI 목업은 화면 배치와 시각 표현을 참고하기 위한 자료다. Guest·Member 권한, 방장·관전자 상태, 투표시간, 렌주, 재접속, 결과·Rating, 멱등성은 서비스 요구사항 및 기능 명세와 구현 계약을 기준으로 구현한다. 목업 이미지가 공식 묶음에 없다는 사실은 MVP 기능 범위를 변경하지 않는다.

### 3.2 AI 개발 도구와 ANALYSIS 런타임의 경계

Codex 같은 AI 개발 도구로 일반 애플리케이션 코드를 생성·수정하는 것은 허용하며, 서비스 플랫폼 담당자가 결과를 검토하고 인수·통합한다. 이는 사용자에게 판세 분석 결과를 제공하는 ANALYSIS 런타임과 다르다. MVP에서는 분석 Pod·모델/API·Redis Streams 처리·재시도·DLQ·분석 부하 시험을 구현하지 않되, 후속 확장을 위한 이벤트와 식별 계약은 보존한다.

## 4. 원 목표 구조와 MVP 구조 비교

| **구성요소**    | **원 목표 구조**                           | **MVP 구조**    | **판단**                                  | **MVP에서 잃는 검증**          |
|-----------------|--------------------------------------------|-----------------|-------------------------------------------|--------------------------------|
| Physical Server | 4대                                        | 4대             | Failure Domain과 팀 병렬 작업을 위해 유지 | 없음                           |
| 서비스 VM       | 18개                                       | 16개            | lb-02·maxscale-02만 제외                  | LB·MaxScale 인스턴스 Failover  |
| VRouter         | 4대                                        | 4대             | Server별 사설망·정적 라우팅 유지          | 없음                           |
| Control Plane   | 3대                                        | 3대             | stacked etcd quorum 검증 유지             | 없음                           |
| Worker          | 2대                                        | 2대             | 재스케줄·Replica·HPA 검증 유지            | 없음                           |
| LB              | HAProxy+Keepalived 2대                     | HAProxy 1대     | Endpoint 유지, 자동 VIP 전환 후속         | VRRP·GARP·VIP Failover         |
| MariaDB         | Primary+Replica                            | Primary+Replica | 영속성·복제·승격·RPO 유지                 | 없음                           |
| MaxScale        | 2대                                        | 1대             | DB Stable Endpoint와 Routing 역할 유지    | Proxy Instance 우회            |
| Redis           | 1 Workload+AOF/PVC                         | 동일            | Runtime State·복구 경계 유지              | Sentinel/Cluster는 원래 비채택 |
| ANALYSIS        | 독립 Workload                              | 미배포          | 게임 비권위 보조 기능을 후속 확장         | 분석 처리·지연·stale·부하      |
| CI/CD·GitOps    | Jenkins·Harbor·Argo CD                     | 동일            | Build와 CD 책임 경계 유지                 | 도구 자체 HA                   |
| 관측성          | Prometheus·Grafana·Loki·Alloy·Alertmanager | 동일            | 검증·Timeline·E-mail 알림 유지            | 장기·HA 저장                   |

> **수량 판정** Server-01 4VM + Server-02 4VM + Server-03 5VM + Server-04 3VM = MVP 서비스 VM 16개다. loadgen은 5번째 Windows Host의 외부 지원 VM이며 서비스 VM 수에 포함하지 않는다.

## 5. MVP 물리·네트워크·Endpoint 구조

| **Physical Server** | **수동 VM 생성 담당** | **MVP VM**                                       | **Failure Domain 역할**     |
|---------------------|-----------------------|--------------------------------------------------|-----------------------------|
| Server-01           | 정태훈                | VRouter-01, CP-01, Worker-01, MariaDB-02 Replica | CP·Worker·Replica 분산      |
| Server-02           | 김상희                | VRouter-02, CP-02, Worker-02, MariaDB-01 Primary | CP·Worker·Primary 분산      |
| Server-03           | 최유준                | VRouter-03, LB-01, CP-03, MaxScale-01, Harbor    | Endpoint·DB Access·Registry |
| Server-04           | 이유빈                | VRouter-04, NFS, Ansible Controller              | Storage·Automation·Network  |

| **영역**       | **주소·구성**                                 | **MVP 기준**                                            |
|----------------|-----------------------------------------------|---------------------------------------------------------|
| 외부 L2        | 10.1.93.0/24 · VMnet0 Bridged                 | Windows Host의 물리 NIC에 고정한 외부망                 |
| Host / VRouter | Host .70/.72/.74/.76, VRouter .71/.73/.75/.77 | Host와 Router 주소를 분리                               |
| LB 관리 주소   | LB-01 10.1.93.78                              | Bridged 관리·서비스 VM 주소; Common Endpoint .90과 구분 |
| 사설망         | 192.168.51.0/24~192.168.54.0/24               | Server별 Host-only, VMware DHCP 비활성화                |
| vNIC           | VRouter 2장, 일반 내부 VM 1장                 | VRouter는 Bridged+Host-only, 일반 VM은 Host-only        |
| 사설망 통신    | VRouter 정적 라우팅                           | NAT 없이 Source IP 보존                                 |
| 외부 통신      | Masquerade                                    | Private Subnet에서 외부로 나갈 때만 적용                |
| Pod Network    | Calico VXLAN                                  | Node 간 VM IP·정적 Route 위에서 동작                    |
| 외부 지원      | Windows Host .92 / loadgen VM .91             | 부하·Backup·Raw Evidence; 서비스 Failure Domain 밖      |
| 예약·미사용    | 10.1.93.93~99 예약, 6번째 장비 미사용         | 확장 전까지 서비스 구성요소를 배치하지 않음             |

각 Physical Server는 Intel Core i5-13400 10 Core/16 Thread, RAM 64GB를 기준으로 하며, 각 Windows Host의 C 드라이브에는 프로젝트 VM 공간 500GB를 확보하고 Virtual Disk는 Thin Provisioning으로 생성한다. 실제 Interface 이름과 Host-only VMnet 번호는 추측하지 않고 VMware·CentOS에서 확인한 뒤 Inventory·host_vars·Runbook에 고정한다.

| **Server** | **MVP VM 기준 vCPU** | **RAM** | **Virtual Disk** | **수용성 판단**                                         |
|------------|----------------------|---------|------------------|---------------------------------------------------------|
| Server-01  | 15                   | 45GB    | 310GB            | Worker와 DB Replica 경합을 관측하면서 기존 시작값 유지  |
| Server-02  | 15                   | 45GB    | 310GB            | Worker와 DB Primary 경합이 성능 결과에 미치는 영향 확인 |
| Server-03  | 11                   | 19GB    | 280GB            | Harbor I/O와 단일 LB·MaxScale·CP 영향 분리              |
| Server-04  | 5                    | 13GB    | 260GB            | lb-02·maxscale-02 제거분을 NFS·Backup 여유로 보존       |

Worker의 8 vCPU·28GB RAM·140GB Disk 시작값은 ANALYSIS 미배포로 여유가 생기더라도 줄이지 않는다. 남은 Worker 한 대가 필수 Runtime을 수용하는 장애 조건과 향후 원 목표 확장 여유를 보존하기 위한 값이며, 실제 requests/limits는 부하 시험 후 동결한다.

### 5.1 Common Endpoint

| **주소**          | **사용 주체**          | **전달 대상**    | **MVP 한계**                        |
|-------------------|------------------------|------------------|-------------------------------------|
| 10.1.93.90:80/443 | 사용자·loadgen         | Gateway NodePort | LB-01 장애 시 자동 우회 없음        |
| 10.1.93.90:6443   | 관리자·Node·Ansible    | CP-01~03         | 주소는 유지하되 LB가 SPOF           |
| 10.1.93.90:3306   | Backend·제한 관리 경로 | MaxScale-01      | MaxScale-01 장애 시 Proxy 우회 없음 |

MVP에서 10.1.93.90은 LB-01이 고정 보유하는 공통 서비스 주소다. Keepalived·VRRP 자동 전환은 구현하지 않으므로 이를 HA VIP로 과장하지 않는다. 원 목표 확장 시 같은 주소를 Keepalived 소유권 전환 대상으로 바꾸며 Application Endpoint는 유지한다.

## 6. MVP 서비스 기능·상태 계약

| **영역**    | **Must 계약**                                                     | **서버 권위·완료 조건**                                       |
|-------------|-------------------------------------------------------------------|---------------------------------------------------------------|
| 사용자      | Guest 임시 세션, Member 회원가입·로그인·로그아웃                  | Guest는 방 생성·영구 전적 금지, Member만 방 생성·Rating 반영  |
| 입력 검증   | 회원가입·방 생성·채팅의 필수 길이·형식·허용 범위                  | 클라이언트 표시와 별개로 서버가 거부하고 상태를 변경하지 않음 |
| 방          | 공개·비공개·비밀번호, 최소 Ready·최대 인원, 투표시간 5/10/15/30초 | 기본 15초, 방장 변경 시 모든 Ready 해제                       |
| 방장        | 게임 시작·설정 변경, 이탈 시 Member 승계                          | 입장 순서가 가장 빠른 연결 Member 승계, 없으면 방 종료        |
| 게임 시작   | 흑·백 각 Ready 1명 이상 + 최소 Ready + 방장 Start                 | 게임 시작 순간 Ready 참가자를 현재 판 참가자로 고정           |
| 투표        | 현재 턴 팀 1인 1표, 마감 전 변경, 유효 좌표·deadline 검사         | 이탈·단절 시 표 즉시 제거, 재접속해도 이전 표 미복원          |
| 착수        | 최고 득표 좌표, 동률은 서버 무작위, 공식 Move 1개                 | game_id+turn_no당 Move 최대 1개, Pass는 Move 없음             |
| 렌주        | 15×15, 흑 정확히 5목·3-3·4-4·장목 금수, 백 5목 이상 허용          | 금수·승패 판정은 서버 권위                                    |
| 무투표·이탈 | 한 팀 0표 Pass, 상대 팀도 연속 0표면 공동 패배                    | 인프라 장애와 사용자 무투표·몰수를 분리                       |
| 재접속      | 30초 유예와 Room·Team·Game 상태 복원                              | 복구 불가 시스템 장애는 SYSTEM_INVALID, Rating 미반영         |
| 결과        | 승·패·무·몰수·공동 패배·시스템 무효, 결과 화면·대기방 복귀·나가기 | game_id당 Result·전적·Rating 최대 1회                         |
| Rating      | Member 초기 1000, 팀 평균 Elo, K=32; Guest는 가상 1000            | Member만 영구 반영하고 랭킹은 Rating 기준으로 정렬            |

### 6.1 영속성과 Runtime State

- MariaDB는 Member·MemberStats·Move·GameResult·RatingHistory 등 영속 원본을 소유한다.

- Redis는 Session·Room·Participant·Ready·Game·Turn·현재 표 등 Runtime State와 재접속 Snapshot을 담당한다.

- Redis AOF/PVC 장애 시 MariaDB에서 Board·종료 Game 등 파생 가능한 범위와 복구할 수 없는 활성 상태를 구분한다.

- 시스템 장애를 사용자 이탈로 계산해 몰수패·공동 패배·Rating을 생성하지 않는다.

- ANALYSIS는 MVP에서 실행하지 않지만 공식 Move 직후의 Board Snapshot과 game_id·move_no·MOVE_APPLIED 이벤트 계약은 원 목표 확장을 위해 보존한다.

- 후속 ANALYSIS는 흑·백 예상 승률을 산출하되 game_id+move_no가 현재 보드와 일치할 때만 표시한다. stale 결과는 폐기하고 분석 지연·실패가 다음 Turn이나 게임 권위 판정을 막지 않도록 한다.

## 7. 구성요소별 유지·단일화·제외 판단

| **구성요소** | **MVP 결정** | **선택 이유**                             | **대안·비선택 이유**                    | **확장 방법**                    |
|--------------|--------------|-------------------------------------------|-----------------------------------------|----------------------------------|
| CP 3         | 유지         | quorum·CP 장애 검증이 핵심                | 1대 축소는 검증축 소실                  | 변경 없음                        |
| Worker 2     | 유지         | 재스케줄·Replica·HPA 필요                 | 1대는 연속성 검증 불가                  | 변경 없음                        |
| LB           | 1대로 단일화 | Endpoint 역할은 유지하고 VRRP 복잡도 절감 | LB 제거는 API·서비스·DB Endpoint 재설계 | lb-02·Keepalived 추가            |
| MariaDB 2    | 유지         | 복제·승격·RPO·영속성 검증                 | 단일 DB는 핵심 데이터 검증 약화         | 변경 없음                        |
| MaxScale     | 1대로 단일화 | Stable DB Endpoint와 Routing 역할 유지    | Backend 직결은 후속 삽입 비용 발생      | maxscale-02·HAProxy Backend 추가 |
| Redis        | 유지         | Runtime State·재접속·다중 Backend 일관성  | Process Memory는 장애 복구 불가         | AOF/PVC 유지, Cluster는 비채택   |
| ANALYSIS     | 원 목표 확장 | 게임 권위와 무관한 보조 기능              | MVP 병행은 코드·부하·검증량 증가        | 독립 Consumer Workload 추가      |
| NFS          | 유지         | Redis·Jenkins PVC와 복구 시험             | 임시 Local 저장은 Migration 유발        | SPOF를 정직하게 유지             |
| CI/CD·GitOps | 유지         | 자동 Build·Registry·Desired State 검증    | 수동 배포는 핵심 주장 소실              | 고도화 Promotion만 후속          |
| 관측성       | 유지         | 장애 Timeline과 Evidence 확보             | 수동 모니터링은 검증 재현성 부족        | HA·장기 보존은 후속              |

**Kubernetes 연속성.** Control Plane 3대는 stacked etcd quorum과 단일 Control Plane 장애 시 API 연속성을 검증하기 위한 최소 수량이다. 1대로 줄이면 설치는 단순해지지만 프로젝트의 HA 검증축이 사라진다. Worker 2대도 Pod 재스케줄, Replica 분산, HPA와 단일 Worker 장애 후 남은 자원 수용을 확인하는 최소 조건이므로 유지한다.

**공통 Endpoint.** LB를 없애고 각 서비스에 직접 연결하면 API·서비스·DB의 외부 접근 계약이 갈라지고 원 목표 확장 때 Application 설정을 다시 바꿔야 한다. 따라서 HAProxy 1대는 유지하되 Keepalived·VRRP와 두 번째 LB만 후속으로 미뤄 Endpoint 계약과 구현량을 함께 통제한다.

**영속 데이터와 DB 접근.** MariaDB Primary·Replica는 복제, 승인된 승격, RTO/RPO와 데이터 무결성을 검증하는 핵심 구조다. MaxScale도 Backend 직결보다 Stable DB Endpoint와 topology routing 경계를 보존하므로 1대를 유지한다. 두 번째 MaxScale만 추가하면 연결 문자열을 바꾸지 않고 원 목표 구조로 확장할 수 있다.

**Runtime State와 Storage.** Redis를 Process Memory로 대체하면 다중 Backend의 Room·Game 상태 공유와 재접속 복구를 검증할 수 없다. AOF/PVC는 유지하되 Sentinel/Cluster는 현재 규모에서 추가 운영 복잡도가 크므로 채택하지 않는다. NFS는 SPOF이지만 Redis·Jenkins PVC와 Restore 실험의 공통 기반이므로 유지하며, 임시 Local Storage로 바꿔 이후 Migration을 만드는 대신 장애 영향과 복구 한계를 그대로 검증한다.

**Delivery와 관측성.** Jenkins·Harbor·Argo CD를 수동 배포로 대체하면 Build, Image, Desired State와 Rollback의 추적성이 사라진다. Prometheus·Grafana·Loki·Alloy·Alertmanager도 장애 Timeline과 Evidence를 재구성하는 검증 계층이므로 유지한다. 대신 도구 자체 HA, 장기 저장, 선택적 Discord 연계만 뒤로 미룬다.

**ANALYSIS 경계.** 판세 분석은 게임 권위와 분리된 보조 Consumer이므로 MVP에서 실행하지 않아도 투표·착수·결과의 핵심 가치는 유지된다. 이벤트·Board Snapshot·game_id+move_no 계약을 먼저 보존해 두면 독립 Consumer를 추가할 수 있으며, 분석 실패·지연·stale 결과가 게임 진행을 막지 않는 원 목표 구조로 확장할 수 있다.

## 8. 구현 순서와 First Success Milestone

| **Milestone**             | **선행 조건**            | **완료 상태**                                                      | **다음 단계가 얻는 것**           |
|---------------------------|--------------------------|--------------------------------------------------------------------|-----------------------------------|
| M0 수동 공통 기반         | Windows Host·VMware 준비 | 16VM·loadgen 생성, CentOS·SSH·계정·Disk·vNIC 확인                  | Ansible과 역할별 병렬 작업 시작   |
| M1 Foundation             | M0                       | Inventory, Guest Baseline, 시간·Repository·Firewall 공통 상태 일치 | Network·Platform·Data 자동화 기반 |
| M2 Network                | M1                       | VRouter·정적 Route·Return Path·DNS/NTP·필수 Port 정상              | Kubernetes·DB·Storage 병렬 구축   |
| M3 Platform/Data          | M2                       | CP 3·Worker 2·Calico, MariaDB·MaxScale·NFS·Redis PVC 정상          | 서비스 실행 기반                  |
| M4 Delivery/Observability | M3 일부 + TLS            | Harbor·Jenkins·Argo CD·Metric·Log·E-mail 연결                      | 배포·검증 경로                    |
| M5 First Success          | M3·M4                    | Guest/Member → 방 → 투표 → Move → Result → 대기방 E2E 1회 성공     | 기능 안정화와 부하·장애 Baseline  |
| M6 MVP Complete           | M5                       | Must 기능·검증·Evidence Gate 통과                                  | Go/No-Go와 Technical Freeze       |

### 8.1 병렬화 원칙

- Network와 Guest Baseline이 닫히면 Kubernetes, DB·Storage, CI/CD·관측성을 서로 기다리지 않고 병렬 진행한다.

- Application은 외부 계약과 Mock 설정을 먼저 준비하고 Gateway·Redis·DB Endpoint가 열리면 통합한다.

- 각 담당자는 자신의 Ansible Role과 Component Validation을 함께 만든다.

- 통합 Gate에서만 Provider와 Consumer가 공동 확인하고, E2E·장애·복구·부하는 네 명이 함께 수행한다.

## 9. 핵심 검증과 Evidence 설계

| **검증축**    | **MVP Test**    | **핵심 측정값**                                   | **성공 판단**                                                                      | **확장 여부**          |
|---------------|-----------------|---------------------------------------------------|------------------------------------------------------------------------------------|------------------------|
| Ansible       | Q-01~03         | 구축·재실행 시간, changed/failed, 개입·Drift      | 동일 완료 Gate에서 수동 대비 자동화 효과와 멱등성 확인                             | 유지                   |
| 실시간 권위   | G-08·RT-01      | Move·Result·Rating 중복, stale 요청, 상태 일치    | Move 최대 1개, Result·Rating 1회, Pass Move 0개                                    | ANALYSIS 조건만 후속   |
| Backend 복구  | RT-02           | 복구시간, Snapshot 일치, 오패배 건수              | 상태 복원 또는 SYSTEM_INVALID, 장애 원인 오패배 0건                                | 유지                   |
| Control Plane | F-02            | etcd quorum, API 성공, 복구시간                   | CP 1대 중단에도 2/3 quorum·API 유지                                                | 전체 Restore는 Should  |
| Worker·HPA    | F-03·L-04·Q-08  | 재스케줄, 실패 요청, RPS·p95/p99·Replica          | 남은 Worker 자원 조건에서 서비스 복구, 동일 부하 비교                              | 유지                   |
| 단일 LB       | F-01 MVP 변형   | 감지시간, 4개 Listener 영향, 수동 복구시간        | 자동 우회를 기대하지 않고 중단 범위·감지·복구를 포트별로 증명                      | VRRP Failover는 후속   |
| 단일 MaxScale | F-05 MVP 변형   | DB Endpoint 오류, Connection 영향, 수동 복구시간  | Proxy HA와 DB 복제를 구분하고 복구 후 신규 연결·Write 경로 확인                    | Pair 우회는 후속       |
| DB            | F-06·Q-06       | Write 재개, GTID, ACK 손실, RTO/RPO               | Fencing·승인 후 승격, Split Brain·중복 Write 없음                                  | MaxScale Pair는 후속   |
| Redis         | F-07·DR-03      | AOF/PVC, Snapshot, 중복·유실                      | 권위 DB와 일치, 복구 불가 활성 상태 과장 금지                                      | ANALYSIS Stream은 후속 |
| NFS·DR        | F-09·DR-01·04   | I/O·Checksum·Restore 시간·무결성                  | MariaDB 표본·PVC 파일·UID/GID·Checksum 일치                                        | 유지                   |
| CI/CD         | G-06·Q-07       | Build·Push·Sync·Rollback 시간, 수동 단계          | Commit→Digest→Healthy·Rollback 연결                                                | Promotion 고도화 후속  |
| 관측성        | G-07·F-12/13    | Metric·Log·Alert 시각, E-mail 통보                | 장애 Timeline 재구성, firing/resolved 확인                                         | Discord·HA 저장 후속   |
| Physical FD   | FD-02·FD-03 MVP | API·Pod·GTID·Endpoint·Harbor·오패배·복구 Timeline | Server-02의 Runtime·DB 복합 장애와 Server-03의 단일 Endpoint 집중 영향을 각각 분리 | FD 전체 Matrix 후속    |

원 목표의 F-01과 F-05는 이중 LB·MaxScale의 자동 우회를 전제로 하지만 MVP에서는 같은 성공 기준을 사용할 수 없다. MVP 변형 시험은 장애를 숨기지 않고 감지시간, 포트별 영향, 수동 복구시간과 복구 순서를 측정한다. FD-03은 LB-01·MaxScale-01·CP-03·Harbor가 함께 중단되는 MVP의 핵심 복합 장애이므로 FD-02와 함께 대표 시험으로 수행한다.

### 9.1 Evidence 원칙

- 모든 시험은 Run ID, 시작·주입·복구·완료 시각과 동일 기준시계를 사용한다.

- TXT·JSON·CSV·Log·YAML Export를 우선하며 Screenshot은 보조 증거로 사용한다.

- Secret·Token·Private Key·Password는 Evidence에 남기지 않는다.

- 측정값은 실제 환경에서 얻은 값으로 동결하며 사전 문서의 시작값을 성과로 쓰지 않는다.

- Raw Evidence는 외부 loadgen 별도 Disk에 보존하고 Git에는 요약·Checksum·위치를 기록한다.

## 10. WBS·선후관계·Critical Path

| **WBS** | **작업 묶음**                              | **Provider**             | **선행**        | **Consumer·후속**         | **일정 경로**     |
|---------|--------------------------------------------|--------------------------|-----------------|---------------------------|-------------------|
| W-01    | 수동 VM·CentOS·SSH 기반                    | 각 Server 담당           | 없음            | 전 역할                   | 주 경로           |
| W-02    | Inventory·Guest Baseline                   | 이유빈 + 각 Domain Owner | W-01            | Network·K8s·Data·Delivery | 주 경로           |
| W-03    | VRouter·Route·Firewall·DNS/NTP             | 이유빈                   | W-02            | 전 역할 통신              | 주 경로           |
| W-04    | LB-01·Common Endpoint·TLS                  | 이유빈                   | W-03            | K8s API·서비스·DB 접근    | 주 경로           |
| W-05    | kubeadm·Calico·Gateway·Add-on              | 정태훈                   | W-03·W-04       | Application·CI/CD·관측성  | 주 경로           |
| W-06    | MariaDB·MaxScale·NFS·Backup                | 김상희                   | W-03            | Application·K8s PVC·DR    | 병렬·M5 합류      |
| W-07    | Harbor·Jenkins·Argo CD                     | 최유준·정태훈            | W-03·W-05 일부  | Application 배포          | 병렬·M4 합류      |
| W-08    | Prometheus·Loki·Alloy·Grafana·Alertmanager | 최유준                   | W-05            | 검증·Evidence             | 병렬·검증 전 합류 |
| W-09    | Frontend·Backend·Redis·서비스 계약         | 정태훈                   | W-05~08         | E2E                       | 주 경로           |
| W-10    | 기능·장애·복구·부하·Before/After           | 4인 공동                 | W-09            | Freeze·발표               | 주 경로           |
| W-11    | 원 목표 확장                               | 각 Domain Owner          | MVP Complete·Go | 재검증                    | 후속·No-Go 가능   |

> **Critical Path** W-01 수동 기반 → W-02 Guest Baseline → W-03 Routing → W-04 Common Endpoint → W-05 Kubernetes → W-09 Application E2E → W-10 핵심 검증 → Technical Freeze가 주 경로다. W-06 Data·Storage, W-07 Delivery, W-08 Observability는 Network 이후 병렬 진행하되 각각 M5, M4, 검증 시작 전에 합류해야 한다.

일정 연결은 W-01~03을 8월 27~30일, W-04~08을 8월 31일~9월 3일, W-09를 9월 4~10일, W-10을 9월 11~16일에 배치한다. 병렬 작업도 완료 Gate를 넘기지 못하면 합류점 이후 주 경로를 지연시키므로, “필수 작업”과 “현재 시점의 주 경로”를 구분해 관리한다.

## 11. 4인 역할·Provider·Consumer

| **담당자** | **Owner 영역**                                 | **주요 제공물**                                                                | **주요 소비물**                        | **수동 VM 담당** |
|------------|------------------------------------------------|--------------------------------------------------------------------------------|----------------------------------------|------------------|
| 이유빈     | Network·Ansible·External Infra                 | Inventory·공통 변수·VRouter·Route·Firewall·Guest Baseline·통합 실행 프레임워크 | 각 Domain Role·검증 결과               | Server-04        |
| 정태훈     | Kubernetes·서비스 플랫폼·Application 인수/통합 | Cluster·Calico·Gateway·Redis Manifest·Argo CD·Application Desired State        | Network·DB Endpoint·Registry·관측 경로 | Server-01        |
| 김상희     | DB·Storage·Backup/DR                           | MariaDB·MaxScale·NFS·Redis 영속/복구 기준·Backup/Restore                       | Network·K8s PVC 소비 계약              | Server-02        |
| 최유준     | Observability·CI/CD·Test Support               | Jenkins·Harbor·CI·Prometheus·Grafana·Loki·Alloy·Alertmanager                   | Cluster·App Metric/Log·DB/NFS Exporter | Server-03        |

### 11.1 검증 책임

| **검증 수준**      | **책임**            | **예시**                                             |
|--------------------|---------------------|------------------------------------------------------|
| Component          | 해당 Owner          | Route·Cluster·Replication·PVC·Build·Scrape 개별 확인 |
| Integration        | Provider + Consumer | Network↔K8s, K8s↔NFS, App↔DB, Jenkins↔Harbor↔Argo CD |
| E2E·장애·복구·부하 | 4인 공동            | Client부터 Result·Evidence까지 동일 Run ID로 확인    |

역할은 기술 개수로 4등분하지 않는다. 예상 구현시간·Troubleshooting·검증량·통합 부담으로 균형을 판단한다. 이유빈이 모든 Role을 대신 작성하지 않으며 각 Owner가 자신의 Domain Role과 검증 책임을 보유한다. 최유준도 전체 테스트 전담자가 아니다.

| **담당자** | **초기 집중**                             | **중·후반 집중**                   | **상대 부하**             | **지연 영향**                                             |
|------------|-------------------------------------------|------------------------------------|---------------------------|-----------------------------------------------------------|
| 이유빈     | VM 입력·Inventory·Network·Common Endpoint | 통합 자동화·Network 장애 지원      | 초기 매우 높음, 이후 중간 | Routing·Endpoint 지연은 모든 병렬 작업의 시작을 막음      |
| 정태훈     | Cluster·Gateway·Application 계약          | Redis·Argo CD·Application E2E 통합 | 전 기간 높음, 통합기 최고 | Platform 또는 App 계약 지연은 First Success를 직접 지연   |
| 김상희     | MariaDB·MaxScale·NFS                      | Backup/Restore·DB/Redis 복구 검증  | 중반·검증기 높음          | DB Endpoint 지연은 App 통합, DR 지연은 완료 Gate를 막음   |
| 최유준     | Harbor·Jenkins·관측성                     | 부하 지원·Timeline·Evidence        | 중반·검증기 높음          | Delivery 지연은 배포, 관측성 지연은 장애 증거 확보를 막음 |

이 표는 시간 단위 확정치가 아니라 역할 부하의 상대 비교다. 날짜별 개인 Task와 공수는 MVP 실행·협업 실시설계에서 확정하되, 초기 Network 병목과 통합기 서비스 플랫폼 집중을 먼저 완화하도록 공동 지원 순서를 정한다.

## 12. 실행 일정·동결·Go/No-Go

| **기간**         | **목표**                                                                                           | **완료 기준**                                                          |
|------------------|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| 8월 24~26일      | 확장 호환형 MVP 도출 검토 → 최종 프로젝트 기획안 통합 → 실행·협업 실시설계 → 협업 환경·GitHub 구성 | 각 선행 산출물을 확인한 뒤 다음 단계에 착수하고 첫 구현 Task 입력 확보 |
| 8월 27~30일      | VM·Network·SSH·Ansible Baseline                                                                    | M0~M2 Gate                                                             |
| 8월 31일~9월 3일 | Kubernetes·DB/Storage·CI/CD·관측성 병렬 구축                                                       | M3·M4 Component Gate                                                   |
| 9월 4~6일        | Application·Redis·DB·Gateway·GitOps 통합                                                           | First Success 1회                                                      |
| 9월 7~10일       | E2E 안정화·Must 기능 종료                                                                          | MVP Functional Freeze                                                  |
| 9월 11~14일      | 장애·복구·부하·Before/After 검증                                                                   | 핵심 Evidence 확보                                                     |
| 9월 15일         | 원 목표 확장 Go/No-Go                                                                              | 확장 또는 MVP 확정 결정                                                |
| 9월 16일         | Technical Freeze                                                                                   | 완료·검증된 구조만 동결                                                |
| 9월 17~23일      | 문서·발표·Demo·리허설·질의 대비                                                                    | 실제 결과와 증거 반영                                                  |

### 12.1 협업 운영

- 팀은 같은 공간에서 하루 8시간 작업하므로 매일 15분 정례 Integration 회의는 두지 않는다.

- 문제가 생기면 즉시 대면 협의하고 공통 IP·Port·Variable·Schema·Interface 변경만 기록한다.

- Integration Gate에서 짧게 공동 확인하며 필요 시 주간 확인은 5~10분 이내로 한다.

- Technical Freeze 이후에는 치명적 Bug Fix 외 신규 기능·구조 변경을 중단한다.

## 13. MVP 완료 Gate

| **Gate**      | **필수 완료 조건**                                                       | **실패 판정**                                       |
|---------------|--------------------------------------------------------------------------|-----------------------------------------------------|
| 기능          | Guest/Member 진입부터 방·Ready·투표·Move·Result·대기방 복귀까지 E2E 동작 | Must 흐름 단절 또는 서버 권위 위반                  |
| 상태·데이터   | 멱등성·재접속·오패배 방지·Member 영속·Guest 임시 경계 확인               | 중복 Move/Result/Rating, 상태 불일치, 장애 오패배   |
| 인프라        | 16VM, CP3·Worker2, LB1·MaxScale1, DB2, Redis·NFS·CI/CD·관측성 연결       | Common Endpoint 또는 핵심 Dependency 불통           |
| 자동화        | 수동 기반 이후 주요 Role 실행, 재실행·부분 실패·Drift 복원 증거          | Playbook 0만 있고 Component/Integration Gate 미통과 |
| 검증·Evidence | 핵심 Test 완료, 실제 측정값·Raw Evidence·Run ID·Checksum 확보            | 근거 없는 수치, 필수 시험 또는 원본 증거 누락       |

### 13.1 Functional Freeze

- Client → 10.1.93.90 → Gateway → Frontend/Backend → Redis·MariaDB 경로가 연결된다.

- Git → Jenkins → Harbor → GitOps → Argo CD → Kubernetes 경로가 연결된다.

- Prometheus → Alertmanager → E-mail 알림 경로가 연결된다. Prometheus → Grafana의 Metric 조회와 Alloy → Loki → Grafana의 Log 조회 경로가 연결된다. Discord는 선택적으로만 추가한다.

- 핵심 게임과 재접속·결과 처리가 통합 환경에서 반복 동작한다.

Should와 Could 항목의 미완료는 MVP 완료를 막지 않는다. Must 기능 또는 핵심 검증이 실패하면 화면 시연이 가능해도 MVP 완료로 판정하지 않는다.

## 14. 원 목표 구조 확장 경로

| **확장 대상** | **MVP**                 | **원 목표**            | **추가 작업**                                                                           | **Application·Data 영향**                  |
|---------------|-------------------------|------------------------|-----------------------------------------------------------------------------------------|--------------------------------------------|
| LB            | lb-01·.90 고정          | lb-01/02 + Keepalived  | Server-04에 lb-02(.79), VRRP·GARP·Failover와 HAProxy 동기화                             | Endpoint 변경 없음, Migration 없음         |
| MaxScale      | maxscale-01             | maxscale-01/02         | Server-04에 maxscale-02(192.168.54.40), HAProxy Backend·Health 추가                     | DB 연결 문자열 변경 없음                   |
| ANALYSIS      | 이벤트·식별 계약만 보존 | 독립 Consumer Workload | 분석 Pod·모델/API, Redis Streams·Consumer Group, 재시도·DLQ, stale 격리, 분석 부하 시험 | 게임 권위 변경 없음, 데이터 Migration 없음 |
| 장애 검증     | 대표 시나리오           | 원 목표 Matrix         | LB·MaxScale Pair, ANALYSIS, 관련 Physical FD 재검증                                     | 기존 Test 확장                             |
| 자동화        | 단일 인스턴스 Role      | HA 인스턴스 포함       | Inventory Host·Variable·Template·Validation 추가                                        | Role 전면 재작성 없음                      |

### 14.1 Go/No-Go 기준

| **판정 질문** | **Go 조건**                                                      |
|---------------|------------------------------------------------------------------|
| MVP 안정성    | Must와 핵심 검증을 완료하고 치명적 상태·데이터 오류가 없다.      |
| Evidence      | 발표 가능한 실제 정량 증거가 확보되었다.                         |
| 남은 시간     | 확장과 재검증을 9월 16일 전에 완료할 수 있다.                    |
| Rollback      | 확장 실패 시 검증된 MVP로 되돌릴 수 있다.                        |
| 작업 부하     | 특정 담당자에게 감당하기 어려운 Critical 작업이 집중되지 않는다. |

> **No-Go 원칙** 조건 하나라도 충족하지 못하면 MVP를 최종 구현 구조로 동결한다. 원 목표 구조를 일부만 추가한 상태를 전체 완료처럼 발표하지 않는다.

## 15. 리스크·제약·주장 경계

| **리스크·제약**                   | **영향**                                      | **완화·기록 원칙**                                                     |
|-----------------------------------|-----------------------------------------------|------------------------------------------------------------------------|
| 단일 LB                           | 서비스·API·DB Common Endpoint 공통 SPOF       | 감지·영향·수동 복구를 측정하고 자동 Failover로 표현하지 않는다.        |
| 단일 MaxScale                     | DB Proxy 장애 시 Stable Endpoint 중단         | DB 복제와 Proxy HA를 구분하고 원 목표 확장으로 넘긴다.                 |
| NFS 1대                           | Redis·Jenkins PVC와 Backup Staging 영향       | 의도적 SPOF로 기록하고 NFS/PVC Restore를 검증한다.                     |
| Harbor·Ansible Controller 단일 VM | 신규 Pull/Push·자동화 재실행 지연             | Runtime 즉시 영향과 복구 관리 경로를 분리한다.                         |
| 관측 Local Storage                | Worker 장애 시 과거 Metric·Log 일부 손실 가능 | 장기·HA 보존으로 과장하지 않고 Raw Evidence를 외부 보관한다.           |
| Windows 방화벽 비활성화           | 교육장 Host 보호 계층 제한                    | Guest Firewall·NetworkPolicy를 유지하고 환경 전제로 증적한다.          |
| ANALYSIS 제외                     | AI 판세·분석 부하·stale 시험 미실시           | 이벤트·식별 계약을 보존하고 원 목표 확장으로 명시한다.                 |
| 일정 지연                         | 검증·발표 기간 침식                           | Could→Should→원 목표 확장 순으로 미루고 Must·핵심 Evidence를 보호한다. |

### 15.1 금지하는 주장

- 단일 LB·단일 MaxScale을 HA 완료라고 표현하지 않는다.

- MariaDB 복제를 Backup/Restore 또는 무손실 자동 Failover라고 표현하지 않는다.

- Redis AOF/PVC를 완전한 Runtime State HA라고 표현하지 않는다.

- 외부 지원 Server의 보호 Disk를 Off-site·Site DR·HA Storage라고 표현하지 않는다.

- 계획된 성능·RTO·RPO 수치를 실제 측정 결과처럼 쓰지 않는다.

- 미완료 원 목표 확장과 2차 프로젝트 요소를 현재 구현 성과에 포함하지 않는다.

## 16. 최종 기획안·실시설계 인계 및 완료 판정

이 절은 내부 작업 번호를 넘기는 통제표가 아니라, 확장 호환형 MVP 판단을 전체 프로젝트 설명과 실제 실행 문서로 이어 주는 경계를 정의한다. 최종 프로젝트 기획안은 왜 이 범위와 구조를 선택했는지 설명하고, 실행·협업 실시설계는 누가 어떤 입력을 받아 어떤 절차와 증거로 완료할지를 작업 가능한 수준으로 구체화한다.

### 16.1 최종 기획안으로 인계

| **인계 항목** | **확정 내용**                                                                   |
|---------------|---------------------------------------------------------------------------------|
| 구조 관계     | 원 목표 18VM과 MVP 16VM, lb-02·maxscale-02·ANALYSIS 축소 근거와 확장 경로       |
| 범위          | Must·Should·Could·원 목표 확장·1차 비대상·2차 프로젝트 구분                     |
| 실행          | MVP Architecture, WBS·Critical Path, 4인 Owner와 Provider/Consumer              |
| 일정          | Functional Freeze·Go/No-Go·Technical Freeze와 결과물 기간                       |
| 검증          | 핵심 Test·측정값·Evidence 계획과 실제 결과 구분                                 |
| 한계          | 단일 LB·MaxScale, NFS·Harbor·Controller SPOF, 관측 Local Storage, ANALYSIS 제외 |

### 16.2 MVP 실행·협업 실시설계로 인계

| **후속 상세화 대상** | **이 문서에서 고정한 경계**                                                                              |
|----------------------|----------------------------------------------------------------------------------------------------------|
| Task·Timeline        | 역할별 Task ID, 날짜별 작업, 선행 조건, Done Criteria를 상세화한다.                                      |
| Repository·Directory | Infra·App·GitOps 책임과 실제 파일 소유권을 매핑한다.                                                     |
| 실제 환경 값         | 확인된 Interface·VMnet·MTU·Disk·Port·Secret 전달값을 고정한다.                                           |
| 실행 절차            | Role·Playbook·Manifest·GitOps 적용 순서와 확인 명령·Rollback을 작성한다.                                 |
| Integration Contract | Provider·Consumer 입력·출력·변경 Owner·검증 명령을 작성한다.                                             |
| Test·Evidence        | 장애·복구·부하 절차, Stop Condition, 파일명·보존 경로를 작성한다.                                        |
| 협업 환경            | GitHub Organization·Repository·Project·Issue·PR·Review·CODEOWNERS를 설정하고 Notion과의 역할을 분리한다. |

협업 환경에서는 GitHub를 실제 변경과 Task의 Source of Truth로 사용하고, Notion에는 설명·가이드·회의·발표 자료를 둔다. 동일한 Task를 두 곳에 중복 관리하지 않는다. Repository 개수와 Directory, Issue·PR 전문, 명령어와 상세 Checklist는 이 문서에서 미리 고정하지 않고 실행·협업 실시설계와 협업 환경 구성 단계에서 확정한다.

### 16.3 완료 판정

> **완료 조건** MVP 정의, 범위, 아키텍처, 구현·검증, WBS·역할·일정, 완료 Gate, 원 목표 확장, 최종 프로젝트 기획안과 실행·협업 실시설계 인계가 서로 모순 없이 연결되었음을 검토한 뒤 확장 호환형 MVP 도출을 완료로 판정한다. 상세 명령어·저장 경로·Issue 전문은 의도적으로 후속 실시설계에 남긴다.

이 MVP는 핵심 기술을 대폭 제거한 데모가 아니다. 동일한 서비스·Kubernetes API·DB Endpoint와 데이터 책임을 유지하고, 인스턴스·HA·ANALYSIS만 추가하여 원 목표 구조로 확장할 수 있다. 프로젝트의 최종 가치는 기술 개수가 아니라 동일 조건에서 재현되는 구축, 장애 후 복구, 데이터 무결성, 배포 흐름과 실제 측정 Evidence로 증명한다.
