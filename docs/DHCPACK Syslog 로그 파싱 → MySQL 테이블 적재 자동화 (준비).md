# DHCPACK Syslog 로그 파싱 → MySQL 테이블 적재 자동화 (준비)

본 문서는 **Ubuntu 환경에서 DHCP 서버(dhcpd)가 남기는 syslog 로그**를 **rsyslog → MySQL**로 적재한 뒤,  
`SystemEvents.Message` 컬럼에서 **`DHCPACK on ...` 메시지**를 식별/파싱하여 **별도 테이블로 저장**하는 과정을 정리한 기록입니다.

> 목표: DHCPACK 로그로부터 **할당 시각 / 할당 IP / 클라이언트 MAC / 호스트명 / VLAN ID**를 추출해  
> 향후 **NAC/802.1X 및 RADIUS 데이터와 연계**할 수 있는 형태로 저장

---

## 1. 배경 및 목표

802.1X/NAC 랩에서 단말이 인증을 통과하면, DHCP로 **IP를 할당**받고 VLAN이 결정됩니다.  
이 과정에서 운영 관점으로 남겨야 하는 핵심 정보는 다음과 같습니다.

- IP 주소 할당 시간
- 단말기에 할당된 IP 주소
- IP 주소가 할당된 단말의 MAC 주소
- 호스트 이름
- VLAN ID

이를 위해 아래 전략을 사용합니다.

1. **(적재)** rsyslog의 MySQL 모듈을 이용해 syslog를 MySQL `Syslog.SystemEvents`에 저장  
2. **(식별)** `Message LIKE 'DHCPACK on%'` 로 DHCPACK 이벤트만 필터링  
3. **(파싱)** 문자열 함수로 IP/MAC/호스트/VLAN 추출  
4. **(저장)** 별도 DB(`radius`)의 테이블(`dhcp_log`)에 저장해 추후 활용

---

## 2. 실습/검증 환경

- OS: Ubuntu (랩 VM)
- MySQL: 8.0.x (Ubuntu 패키지)
- Syslog 적재: `rsyslog-mysql`
- DHCP: `isc-dhcp-server (dhcpd)`
- DHCP 로그 식별 기준:
  - 메시지 패턴: `DHCPACK on <IP> to <MAC> (<HOSTNAME>) via ens4.<VLAN>`
  - VLAN 추출 기준: `via ens4.<VLAN>` 의 마지막 숫자

---

## 3. 전체 처리 흐름

```text
[isc-dhcp-server(dhcpd)]
        │  (syslog)
        ▼
[rsyslog]
        │  (rsyslog-mysql)
        ▼
[MySQL: Syslog.SystemEvents]
        │  (식별: Message LIKE 'DHCPACK on%')
        ▼
[파싱: IP / MAC / HOSTNAME / VLAN / Time]
        │
        ▼
[MySQL: radius.dhcp_log (별도 테이블)]
```

---

## 4. rsyslog → MySQL 적재 준비

### 4.1 rsyslog-mysql 설치

```bash
sudo apt install rsyslog-mysql
```

<img width="834" height="88" alt="01" src="https://github.com/user-attachments/assets/7142a25b-da95-4b23-be3f-bc49f0e799c5" />


설치 후 MySQL 내부 계정 및 Syslog DB 생성 여부를 확인했습니다.

### 4.2 MySQL 사용자/DB 확인

```sql
USE mysql;
SELECT user FROM user;
SHOW DATABASES;
```

<img width="687" height="391" alt="02" src="https://github.com/user-attachments/assets/049f3841-fe66-4a89-a346-63bfce227a0e" />


<img width="265" height="262" alt="03" src="https://github.com/user-attachments/assets/a23d132d-df78-4a4e-99b8-2b0a20ba8895" />


> `rsyslog` 계정이 생성되어 있고, `Syslog` 데이터베이스가 생성된 것을 확인.

### 4.3 rsyslog 계정으로 접속 및 Syslog 테이블 확인

```bash
mysql -u rsyslog -p
```

<img width="805" height="367" alt="05" src="https://github.com/user-attachments/assets/005161bc-1ab2-481c-80d1-8105a4b77a70" />


```sql
SHOW DATABASES;
USE Syslog;
SHOW TABLES;
```

<img width="270" height="221" alt="06" src="https://github.com/user-attachments/assets/ee2a6acd-9d48-4a3a-b7b2-a27e12f9388b" />


<img width="693" height="315" alt="07" src="https://github.com/user-attachments/assets/fb2c0799-a9e1-4e70-8639-3e072706d766" />


---

## 5. DHCP 로그 식별

### 5.1 DHCP 서버 재기동 후 syslog 발생 확인

```bash
sudo systemctl restart isc-dhcp-server
```

재기동 로그에서 `ens4.<VLAN>` 인터페이스로 리슨 중인 상태를 확인했습니다.

<img width="811" height="610" alt="08" src="https://github.com/user-attachments/assets/8079ab99-d2bc-450f-ad94-12fdf74a2743" />


### 5.2 MySQL(SystemEvents)에서 dhcpd 로그 조회

`SysLogTag`를 기준으로 dhcpd 프로세스 로그를 필터링했습니다.

```sql
SELECT DeviceReportedTime, FromHost, SysLogTag, Message
FROM SystemEvents
WHERE SysLogTag='dhcpd[3195]:'
ORDER BY ID DESC
LIMIT 8;
```

<img width="902" height="601" alt="09" src="https://github.com/user-attachments/assets/b5740b0d-84b3-4bc3-a5fb-3078618aef86" />


### 5.3 DHCP 메시지 패턴 확인

```sql
SELECT Message
FROM SystemEvents
WHERE Message LIKE 'DHCP%';
```

<img width="895" height="662" alt="10" src="https://github.com/user-attachments/assets/a8a6a28d-9677-4dbb-8e5e-4656ccbddae2" />


여기서 최종적으로 사용할 메시지는 다음 패턴입니다.

- `DHCPACK on <IP> to <MAC> (<HOSTNAME>) via ens4.<VLAN>`

```sql
SELECT Message
FROM SystemEvents
WHERE Message LIKE 'DHCPACK on%';
```

<img width="866" height="635" alt="11" src="https://github.com/user-attachments/assets/da15c0ca-c847-49f9-81a8-84835cd50f87" />


---

## 6. DHCPACK 메시지 파싱

> 아래 파싱은 **MySQL 문자열 함수**로만 수행했습니다. (외부 스크립트 없이 DB 내부에서 검증 가능)

### 6.1 IP 주소 추출

```sql
SELECT
  SUBSTRING_INDEX(SUBSTR(Message, LOCATE('DHCPACK on', Message) + 11), ' ', 1) AS ipv4_addr
FROM SystemEvents
WHERE Message LIKE 'DHCPACK on%';
```

<img width="905" height="646" alt="13" src="https://github.com/user-attachments/assets/47e36865-e0c8-4a97-9d97-7afdf830b2c3" />


> `DHCPACK on` 바로 뒤(공백 포함)부터 시작해서 첫 번째 공백 전까지를 IP로 취급

---

### 6.2 MAC 주소 추출

```sql
SELECT
  SUBSTRING_INDEX(SUBSTR(Message, LOCATE('to', Message) + 3), ' ', 1) AS mac_addr
FROM SystemEvents
WHERE Message LIKE 'DHCPACK on%';
```

<img width="905" height="609" alt="14" src="https://github.com/user-attachments/assets/d65f2ca7-4608-467e-8e12-14dae73c425d" />


---

### 6.3 VLAN ID 추출

메시지 마지막이 `via ens4.<VLAN>` 형태이므로, **점(.) 기준 마지막 토큰**을 VLAN으로 사용했습니다.

```sql
SELECT
  SUBSTRING_INDEX(Message, '.', -1) AS vlan_id
FROM SystemEvents
WHERE Message LIKE 'DHCPACK on%';
```

<img width="917" height="647" alt="15" src="https://github.com/user-attachments/assets/8ad272a4-3fc9-4317-81a4-326d3fa020b8" />

---

### 6.4 호스트 이름(Hostname) 추출

호스트명은 괄호 `(...)` 안에 존재하므로, 괄호 구간을 추출했습니다.  
(괄호가 없는 경우 빈 문자열 처리)

```sql
SELECT
  IF(LOCATE('(', Message)=0, '',
     SUBSTRING_INDEX(SUBSTR(Message, LOCATE('(', Message) + 1), ')', 1)
  ) AS hostname
FROM SystemEvents
WHERE Message LIKE 'DHCPACK on%';
```

<img width="925" height="648" alt="16" src="https://github.com/user-attachments/assets/02bd60ea-6af7-4c4b-be38-4e5d5d97bdad" />


---

### 6.5 파싱 결과 통합 조회(한 번에 검증)

```sql
SELECT
  SUBSTRING_INDEX(SUBSTR(Message, LOCATE('to', Message) + 3), ' ', 1) AS macaddr,
  SUBSTRING_INDEX(SUBSTR(Message, 13), ' ', 1) AS ipaddr,
  SUBSTRING_INDEX(Message, '.', -1) AS vlanid,
  IF(LOCATE('(', Message)=0, '',
     SUBSTRING_INDEX(SUBSTR(Message, LOCATE('(', Message) + 1), ')', 1)
  ) AS hostname,
  DeviceReportedTime AS event_time
FROM SystemEvents
WHERE Message LIKE 'DHCPACK on%';
```

<img width="910" height="653" alt="17" src="https://github.com/user-attachments/assets/c980d48d-f9b8-4a2f-bdaf-052afb6206f0" />


> 참고: `SUBSTR(Message, 13)`은 `"DHCPACK on "`(12글자 + 공백)을 전제로 한 단순화입니다.  
> 메시지 포맷이 달라질 수 있다면 `LOCATE('DHCPACK on', …)` 방식이 더 안전합니다.

---

## 7. 별도 DB/테이블(radius) 생성

파싱 결과를 **별도 DB**로 저장하여, 향후 RADIUS/AAA 데이터와 연계하기 쉬운 형태로 분리했습니다.

### 7.1 radius DB 생성

```sql
CREATE DATABASE radius;
SHOW DATABASES;
```

<img width="405" height="360" alt="18" src="https://github.com/user-attachments/assets/403ed051-6a6a-4668-9d1d-f290cfbfcf17" />


### 7.2 radius 사용자 생성 및 권한 부여

MySQL 8.x 기준으로 **`GRANT ... IDENTIFIED BY ...`는 허용되지 않으므로**, 사용자 생성과 권한 부여를 분리했습니다.

```sql
CREATE USER 'radius'@'localhost' IDENTIFIED BY 'cisco123';
GRANT ALL PRIVILEGES ON radius.* TO 'radius'@'localhost';
FLUSH PRIVILEGES;
```

<img width="682" height="203" alt="19" src="https://github.com/user-attachments/assets/43da4989-cd78-4816-a3f6-f96257dd8ee0" />


### 7.3 (선택) rsyslog 사용자에게 radius DB 접근 권한 추가

`Syslog.SystemEvents`를 읽어와서 `radius.dhcp_log`로 INSERT를 하려면,  
동일 세션에서 두 DB를 모두 접근할 수 있어야 합니다. 따라서 `rsyslog` 계정에도 권한을 부여했습니다.

```sql
GRANT ALL PRIVILEGES ON radius.* TO 'rsyslog'@'localhost';
FLUSH PRIVILEGES;
```

<img width="710" height="142" alt="21" src="https://github.com/user-attachments/assets/537f1be4-45ec-466e-b0c0-d2de482d1986" />


---

## 8. 파싱 결과 적재(INSERT … SELECT)

### 8.1 저장 테이블 생성

```sql
USE radius;

CREATE TABLE `dhcp_log` (
  `id` int(12) NOT NULL AUTO_INCREMENT,
  `logtime` timestamp NULL DEFAULT NULL,
  `ipaddr` varchar(50) NOT NULL DEFAULT '',
  `macaddr` varchar(50) NOT NULL DEFAULT '',
  `hostname` varchar(50) NULL DEFAULT '',
  `vlanid` int(10) NOT NULL DEFAULT '0',
  `message` text NOT NULL,
  PRIMARY KEY (`id`)
);
```

<img width="915" height="311" alt="23" src="https://github.com/user-attachments/assets/464cca80-ac51-4f09-a466-beec6e671426" />


컬럼 확인:

```sql
SHOW COLUMNS FROM dhcp_log;
```

<img width="686" height="314" alt="24" src="https://github.com/user-attachments/assets/aebb0270-2326-4cc7-bf9c-a55958618b46" />


> **PK(id)의 Default가 NULL로 보이는 현상**은 MySQL에서 흔히 보입니다.  
> `AUTO_INCREMENT + PRIMARY KEY` 컬럼은 실제로 `NULL 저장 불가`이며, 값 미지정 시 자동 증가 값이 채워집니다.

---

### 8.2 INSERT … SELECT로 적재

```sql
INSERT INTO radius.dhcp_log(logtime, ipaddr, macaddr, hostname, vlanid, message)
SELECT
  ReceivedAt AS logtime,
  SUBSTRING_INDEX(SUBSTR(Message, 13), ' ', 1) AS ipaddr,
  SUBSTRING_INDEX(SUBSTR(Message, LOCATE('to', Message) + 3), ' ', 1) AS macaddr,
  IF(LOCATE('(', Message)=0, '',
     SUBSTRING_INDEX(SUBSTR(Message, LOCATE('(', Message) + 1), ')', 1)
  ) AS hostname,
  SUBSTRING_INDEX(Message, '.', -1) AS vlanid,
  Message AS message
FROM Syslog.SystemEvents
WHERE Message LIKE 'DHCPACK on%';
```

<img width="909" height="633" alt="26" src="https://github.com/user-attachments/assets/3f6e455d-f5c5-4da6-9de5-af93394eff87" />


> `logtime`은 예시에서는 `ReceivedAt`를 사용했습니다.  
> 운영 설계에서는 **이벤트 발생 시각(DeviceReportedTime)** 과 **수집 시각(ReceivedAt)** 을 분리해 저장하는 것도 권장됩니다.

---

## 9. 검증 쿼리

### 9.1 최신 데이터 5개 확인

```sql
SELECT id, logtime, ipaddr, macaddr, hostname, vlanid
FROM radius.dhcp_log
ORDER BY id DESC
LIMIT 5;
```

### 9.2 특정 MAC 기준 최근 할당 내역

```sql
SELECT *
FROM radius.dhcp_log
WHERE macaddr='50:00:00:08:00:00'
ORDER BY id DESC
LIMIT 20;
```

### 9.3 VLAN별 최근 할당 내역

```sql
SELECT vlanid, COUNT(*) AS cnt
FROM radius.dhcp_log
GROUP BY vlanid
ORDER BY cnt DESC;
```

