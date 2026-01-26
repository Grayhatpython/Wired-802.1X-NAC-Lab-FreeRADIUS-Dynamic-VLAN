# DB 기반 DHCP 주소정보-할당현황 관리 (MySQL Trigger 기반)

> 목적: **DB(radiuѕ DB)를 
> 1) VLAN별 서브넷/옵션 등록
> 2) 서브넷별 IP 풀 자동 생성
> 3) DHCPACK 로그 기반 IP/MAC 매핑 갱신을 **트리거로 자동화**하는 실습 기록입니다.

---

## 1. 구성 요소

- **Syslog DB (SystemEvents)**  
  - rsyslog/mysql로 적재된 시스템 로그 저장
  - DHCP 서버 로그(`DHCPACK on ...`)를 트리거가 파싱

- **radius DB**
  - `dhcp_log`: DHCPACK에서 파싱한 (시간/IP/MAC/호스트/VLAN/원문) 저장
  - `vlan_network_profiles`: VLAN별 서브넷/옵션/할당범위(계산 결과 포함) 저장
  - `subnet_ip_pool`: 서브넷별 IP 목록 + 상태(U/A) + MAC/사용자 매핑

- **DHCP 시스템**
  - 최종적으로는 `dhcpd.conf`, `dhcpd.leases`를 DB 기반으로 갱신(자동화 예정)

---

## 2. 테이블 설계

> 실제 실습에서 사용한 컬럼 기반으로 정리했습니다. (필요 시 타입/제약은 계속 개선 가능)

### 2.1 `dhcp_log` (DHCPACK 파싱 결과 테이블)

- `SystemEvents.message`에서 DHCPACK 패턴을 찾으면
- IP/MAC/hostname/VLAN을 파싱하여 `radius.dhcp_log`에 적재

증거 스냅샷(컬럼 변경 후):

<img width="744" height="305" alt="shot_20260125_105133" src="https://github.com/user-attachments/assets/9e285f99-363f-4d1c-8773-dcec78e2987b" />


### 2.2 `vlan_network_profiles` (VLAN별 서브넷/옵션/할당범위)

- 입력: `network_addr`, `subnet_mask` (필수)
- 계산(트리거): `broadcast_addr`, `gateway_addr`, `start_ip_addr/end_ip_addr`, `start_ip_number/end_ip_number`, `vlan_group_name`
- 옵션: `dns_server`, `ntp_server`, `domain_name`, `default_lease_time`, `max_lease_time` 등

증거 스냅샷(테이블 생성 및 확인):
- CREATE 성공:
- 
<img width="957" height="258" alt="shot_20260125_111541" src="https://github.com/user-attachments/assets/1317617f-9b11-48a1-92ca-8f0b70cf4e8c" />

- DESC:
- 
<img width="935" height="521" alt="shot_20260125_111545" src="https://github.com/user-attachments/assets/e6d43c61-e3b0-43e2-a960-d5867070b550" />


도메인 컬럼 추가 후:

<img width="938" height="534" alt="shot_20260125_114055" src="https://github.com/user-attachments/assets/b7f70221-34aa-44af-804a-78f537f415f4" />


샘플 데이터(여러 VLAN 등록):

<img width="951" height="618" alt="shot_20260125_114637" src="https://github.com/user-attachments/assets/b780dcff-b029-4c22-ae86-654229cf0932" />


### 2.3 `subnet_ip_pool` (서브넷별 IP 주소 목록/상태 관리)

- `vlan_network_profiles` 입력 시, 해당 서브넷의 **전체 IP 목록**을 자동 생성
- `ip_alloc_state`:
  - `A`: 관리용
  - `U`: 사용 가능(미할당)
- `ip_number`(INT UNSIGNED): `INET_ATON(ip_addr)` 값. 비교/범위 조회 최적화

테이블 생성 및 확인:

<img width="958" height="434" alt="shot_20260125_112317" src="https://github.com/user-attachments/assets/f3795d27-3706-4935-9c86-84c4b1c94a73" />


ip_number 컬럼 추가 후:

<img width="921" height="391" alt="shot_20260126_104530" src="https://github.com/user-attachments/assets/cbbac7c9-656f-46f5-a5b9-2aa5d2e3fb16" />


---

## 3. 트리거 설계 (핵심)

### 3.1 VLAN 프로파일 등록 시 자동 계산 (BEFORE INSERT)

- 대상: `vlan_network_profiles`
- 트리거: `bi_vlan_network_profiles`
- 역할:
  - `broadcast_addr`, `gateway_addr` 계산
  - `start_ip_number/end_ip_number` 산출 후 `start_ip_addr/end_ip_addr` 계산
  - `vlan_group_name = VLAN_<id>` 자동 설정

<img width="967" height="306" alt="shot_20260126_100109" src="https://github.com/user-attachments/assets/4feea5a2-288e-44e2-9c5f-f465d3430750" />


검증(수치 → 문자열 변환 확인):

<img width="407" height="364" alt="shot_20260126_100733" src="https://github.com/user-attachments/assets/1a8293d2-1c72-4c0e-bb0c-3cc6779af67e" />


### 3.2 VLAN 프로파일 등록 후 IP 풀 자동 생성 (AFTER INSERT)

- 대상: `vlan_network_profiles`
- 트리거: `ai_vlan_network_profiles`
- 역할:
  - `network_addr ~ broadcast_addr` 전체 IP를 순회
  - 할당 범위 밖은 `A`, 범위 내는 `U`
  - `subnet_ip_pool`에 일괄 INSERT

<img width="970" height="294" alt="shot_20260126_104718" src="https://github.com/user-attachments/assets/4c6c4d4d-8d72-43b2-942d-d31525c43871" />


실행 결과(서브넷 IP 풀 자동 생성):

<img width="949" height="696" alt="shot_20260126_104803" src="https://github.com/user-attachments/assets/7c57cfb4-a9d9-48ab-8d60-0cd799cc32ac" />


### 3.3 DHCPACK 로그 → `dhcp_log` 적재 (SystemEvents 트리거)

- 대상: `SystemEvents`
- 트리거: `ai_systemevents_dhcp_to_radius_dhcp_log`
- 역할:
  - `DHCPACK on <ip> to <mac> (<hostname>) via ensX.<vlan>` 형태를 REGEXP로 매칭
  - 파싱 후 `radius.dhcp_log`에 적재

<img width="963" height="313" alt="shot_20260125_105535" src="https://github.com/user-attachments/assets/fc102c71-e9f1-4324-83fe-a6a9effd5481" />


### 3.4 `dhcp_log` 적재 후 `subnet_ip_pool` MAC 갱신

- 대상: `dhcp_log`
- 트리거: `ai_dhcp_log`
- 역할:
  - `subnet_ip_pool`에서 해당 IP(`ip_number`)가 존재하고 MAC이 비어있으면
  - `mac_addr`를 업데이트하여 “IP ↔ MAC”을 확정

<img width="938" height="132" alt="shot_20260126_111911" src="https://github.com/user-attachments/assets/539c3319-7939-4a37-9471-23ba6a6f1b2b" />


테스트 삽입(예시):

<img width="958" height="85" alt="shot_20260126_112504" src="https://github.com/user-attachments/assets/435384a1-52a2-4c18-a2cc-0ab5d17c9999" />


반영 결과(일부 IP는 A/일부는 U, MAC 매핑 확인):

<img width="954" height="462" alt="shot_20260126_112511" src="https://github.com/user-attachments/assets/93b5d28d-8e45-4f0f-9493-8733f3c381b5" />


---

## 4. DHCP 설정 파일 연계(진행/예정)

현재까지는 **DB(SSOT) → 테이블/트리거 기반 자동화**까지 구현했고,  
다음 단계로 DHCP 서버 설정 파일을 갱신하는 자동화를 붙이는 구조입니다.

- `dhcpd.conf`
  - `vlan_network_profiles`의 옵션(`dns_server`, `domain_name`, `default_lease_time` 등)을 반영
  - `subnet_ip_pool`에서 `MAC`이 매핑된 항목을 `host` stanza로 생성(고정 할당/예약)

- `dhcpd.leases`
  - 실습에서는 “운영 DB 파일을 직접 수정”하기보다는
  - 필요 시 **상태 테이블 기반으로 정책/리포팅** 위주로 운영(권장)

샘플 `dhcp.conf` 확인(옵션 값 예시):

<img width="697" height="442" alt="shot_20260126_120829" src="https://github.com/user-attachments/assets/83a138db-09dd-43a5-a563-c5d9ec3af54f" />


---

## 5. 트러블슈팅 로그 (Problem → Cause → Fix → Verify)

### 5.1 `ALTER TABLE ... DEFAULT`가 안 먹힘
- **원인:** `DEFAULT` 오타(`DEFALUT`)
- **수정:** `DEFAULT`로 수정 후 실행

<img width="892" height="30" alt="shot_20260125_104519" src="https://github.com/user-attachments/assets/63bf35c4-24f9-4dd5-b4d4-521496ba042e" />


### 5.2 테이블/컬럼명에 작은따옴표 사용
- **원인:** MySQL에서 `'...'`는 문자열 리터럴, 식별자는 `` `...` `` 또는 무따옴표
- **수정:** `CREATE TABLE vlan_network_profiles (...)` 형태로 변경

<img width="686" height="455" alt="shot_20260125_111140" src="https://github.com/user-attachments/assets/3ee0a25a-5368-4473-b34a-55d886cc91e5" />


### 5.3 `DEFAULT` 누락으로 CREATE 실패
- **원인:** `ip_alloc_state VARCHAR(1) NOT NULL 'U'` 처럼 `DEFAULT` 누락
- **수정:** `DEFAULT 'U'`로 명시

<img width="958" height="97" alt="shot_20260125_112238" src="https://github.com/user-attachments/assets/352a2487-3912-468c-9148-f4353cef3b36" />


### 5.4 IP를 숫자로 저장/비교해야 하는 이유
- `INET_ATON/INET_NTOA`로 변환한 `ip_number(INT UNSIGNED)`를 두면
  - 범위 조회/정렬이 단순해지고
  - 트리거에서 매칭 조건이 깔끔해짐

<img width="407" height="364" alt="shot_20260126_100733" src="https://github.com/user-attachments/assets/a3998cf8-d6a3-49ae-a727-29113e235044" />


---

## 6. 실행 순서(재현 가이드)

1) VLAN 프로파일 등록 → IP 풀 자동 생성 확인
- `vlan_network_profiles`에 `network_addr/subnet_mask` INSERT
- `subnet_ip_pool`이 자동 생성되는지 확인

2) SystemEvents에 DHCPACK 로그 유입 → `dhcp_log` 적재 확인
- rsyslog 설정 또는 테스트 INSERT로 트리거 동작 확인

