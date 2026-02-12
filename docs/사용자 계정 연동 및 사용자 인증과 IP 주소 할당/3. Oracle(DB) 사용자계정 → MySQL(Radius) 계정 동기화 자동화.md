# **목표**  
> Oracle DBMS(employee 역할)에서 사용자 계정 정보를 읽어서, FreeRADIUS가 사용하는 MySQL DBMS(radius)의 **staging 테이블**로 복제한 뒤  
> `radcheck`, `radusergroup`(필요 시 `radreply` 등)에 반영하여 **인증 계정/부서(VLAN) 연동**을 자동화한다.

---

## 1. 환경

### IP/역할
| 구성요소 | 역할 | IP |
|---|---|---|
| FreeRADIUS Server | 계정 동기화 스크립트 실행, MySQL(radius) 보유 | `10.0.10.11/28` |
| Oracle DB Server | Docker로 Oracle Free(slim) 구동 | `10.0.10.12/28` |
| DSW(L3 Switch) | 게이트웨이 / 라우팅 | `10.0.10.1` |
| Edge Router | NAT(PAT)로 외부 통신 | (랩 외부망) |

### Mermaid 다이어그램 (GitHub 호환)
```mermaid
flowchart LR
  RADIUS["FreeRADIUS Server<br/>10.0.10.11/28"] -->|"SQLNet (TCP 1521)"| DSW["DSW (L3 Switch)<br/>GW: 10.0.10.1"]
  DSW -->|"VLAN / L2"| DB["Oracle DB Server<br/>10.0.10.12/28<br/>Docker: gvenzl/oracle-free:slim"]
  RADIUS -->|"Default GW"| DSW
  DSW -->|"Default Route"| EDGE["Edge Router<br/>NAT PAT"]
  EDGE -->|"Outside"| HOME["Home Router<br/>Internet"]
```

---

## 2. DB 서버(10.0.10.12): Docker Oracle Free(slim) 구성

### 컨테이너 실행
아래 형태로 Oracle Free(slim)을 실행했다.  
- **APP_USER:** `dbadmin`  
- **APP_USER_PASSWORD:** `dbadmin123`

```bash
docker run -d --name account_db \
  -p 1521:1521 \
  -e ORACLE_PASSWORD='cisco123' \
  -e APP_USER='dbadmin' \
  -e APP_USER_PASSWORD='dbadmin123' \
  -v oracle-vol:/opt/oracle/oradata \
  gvenzl/oracle-free:slim
```

📸 스크린샷: 컨테이너 실행  

<img width="958" height="78" alt="02_docker_run_oracle_free" src="https://github.com/user-attachments/assets/7ced2d48-2cfd-425c-8318-b088fc8ba1f9" />


### Listener 확인 (RADIUS 서버에서)
```bash
nc -zv 10.0.10.12 1521
# succeeded! 이면 OK
```

---

## 3. Oracle 스키마 준비 (v_user_account 뷰)

### 3.1 테이블 생성: `user_account`
Oracle에 사용자 정보를 저장할 **기본 테이블**을 생성.

```sql
CREATE TABLE user_account (
  id        VARCHAR2(20)  NOT NULL,
  emp_name  VARCHAR2(40)  NOT NULL,
  password  VARCHAR2(250) NOT NULL,
  deptcode  VARCHAR2(10)  NOT NULL,
  CONSTRAINT pk_user_account PRIMARY KEY (id)
);
```

📸 스크린샷: 테이블 생성  

<img width="551" height="216" alt="01_oracle_create_table_user_account" src="https://github.com/user-attachments/assets/b8556cbc-6480-461d-982a-6ef1e71d10d2" />


### 3.2 뷰 생성: `v_user_account`
`link_account.sh`가 조회할 이름을 `v_user_account`로 맞추기 위해 VIEW를 생성.

```sql
CREATE OR REPLACE VIEW v_user_account AS
SELECT id, emp_name, password, deptcode
FROM user_account;
```

📸 스크린샷: 뷰 생성  

<img width="485" height="138" alt="03_oracle_create_view_v_user_account" src="https://github.com/user-attachments/assets/9d49ef1f-75ed-4165-9e40-adf3770ce27a" />


### 3.3 테스트 데이터 INSERT
```sql
INSERT INTO user_account(id, emp_name, password, deptcode) VALUES ('A10010', 'Minsu',  'Minsu123',  '300');
INSERT INTO user_account(id, emp_name, password, deptcode) VALUES ('A10020', 'Songsu', 'Songsu123', '310');
INSERT INTO user_account(id, emp_name, password, deptcode) VALUES ('A10030', 'Yumi',   'Yumi123',   '320');
INSERT INTO user_account(id, emp_name, password, deptcode) VALUES ('A10040', 'Zoro',   'Zoro123',   '330');
INSERT INTO user_account(id, emp_name, password, deptcode) VALUES ('A10050', 'Nami',   'Nami123',   '340');
COMMIT;
```

📸 스크린샷: 샘플 데이터 INSERT  

<img width="676" height="507" alt="06_oracle_insert_sample_rows" src="https://github.com/user-attachments/assets/35ee590b-68f1-43ab-ac1e-6980298ebe9c" />


---

## 4. TroubleShooting: INSERT가 안 됨 (Tablespace quota)

### 증상
- `CREATE TABLE` / `CREATE VIEW`는 되는데 **INSERT가 실패**
- 오류 메시지:  
  `user DBADMIN has insufficient quota on tablespace USERS;`

📸 스크린샷(에러 장면 일부)  

<img width="641" height="108" alt="05_oracle_insert_error_quota" src="https://github.com/user-attachments/assets/fd029556-d16b-4fb4-9b03-e07f39fda72c" />


### 원인
- 계정(DBADMIN)이 `USERS` tablespace에 대해 **quota(할당량)가 0**이라 DML(INSERT 등)로 세그먼트 확장이 불가능.

### 해결: SYSDBA로 quota 부여
SYS로 접속 후 DBADMIN에 quota를 부여했다.

```bash
sqlplus sys/cisco123@//10.0.10.12:1521/FREEPDB1 as sysdba
```

```sql
ALTER USER DBADMIN QUOTA UNLIMITED ON USERS;
```

📸 스크린샷: quota 부여  

<img width="832" height="392" alt="04_oracle_fix_quota_users_tablespace" src="https://github.com/user-attachments/assets/6aba6746-b49c-49aa-aba6-89e4cbe01039" />


---

## 5. FreeRADIUS 서버(10.0.10.11): MySQL 스테이징/매핑 테이블

### 5.1 dept → VLAN(groupname) 매핑 테이블: `dept_vlan`
```sql
CREATE TABLE dept_vlan (
  deptcode  VARCHAR(10) NOT NULL,
  groupname VARCHAR(64) NOT NULL,
  deptname  VARCHAR(64) NULL,
  PRIMARY KEY (deptcode)
);
```

예시 데이터:
```sql
INSERT INTO dept_vlan VALUES ('300','VLAN_300','Security Dept');
INSERT INTO dept_vlan VALUES ('310','VLAN_310','Development Dept');
INSERT INTO dept_vlan VALUES ('320','VLAN_320','Business Dept');
INSERT INTO dept_vlan VALUES ('330','VLAN_330','Partner Company');
INSERT INTO dept_vlan VALUES ('340','VLAN_340','Guest Company');
```

📸 스크린샷: `dept_vlan` 확인  

<img width="515" height="270" alt="11_mysql_dept_vlan_select" src="https://github.com/user-attachments/assets/7e5abaf4-9849-4c1f-9bd0-c2318cd46cd0" />


### 5.2 스테이징 테이블: **`temp_empolyee`**
> **중요:** 본 문서에서는 스테이징 테이블명을 최종적으로 `temp_empolyee`로 사용한다.  
> (일부 스크린샷에는 `temp_employee`로 보일 수 있으나, 실제 운영/최종 정리는 `temp_empolyee`로 통일)

권장 스키마(예시):
```sql
CREATE TABLE temp_empolyee (
  emp_id          VARCHAR(20)  NOT NULL,
  emp_name        VARCHAR(40)  NOT NULL,
  password        VARCHAR(250) NOT NULL,
  hashed_password VARCHAR(64)  NOT NULL,
  type            CHAR(1)      NOT NULL,
  deptcode        VARCHAR(10)  NOT NULL,
  groupname       VARCHAR(64)  NULL,
  PRIMARY KEY (emp_id)
);
```

---

## 6. link_account.sh (Oracle → MySQL 복제 자동화)

### 6.1 스크립트 개요
`link_account.sh`는 다음 단계를 수행한다.

1. Oracle `v_user_account`에서 계정 목록 조회 (`CHR(9)`로 TAB 구분)
2. 필요 시 인코딩 변환 (예: `EUC-KR → UTF-8`)
3. 평문 비밀번호를 `smbencrypt`로 해시 생성
4. MySQL에 적용할 SQL(`mysql_apply.sql`)을 생성 후 실행  
   - `DELETE FROM temp_empolyee;`  
   - `INSERT INTO temp_empolyee ...`
5. 마지막으로 `shared.sql`(공통 연동 SQL)을 실행해 실제 반영 (`radcheck`, `radusergroup`)

📸 스크린샷: 헤더/환경변수  

<img width="940" height="617" alt="16_link_account_sh_header_env" src="https://github.com/user-attachments/assets/2e469ea0-af8b-46a4-a5cd-8992b5996f30" />

📸 스크린샷: Oracle SELECT 부분  

<img width="961" height="595" alt="17_link_account_sh_oracle_query" src="https://github.com/user-attachments/assets/2a8a298f-68f4-472c-a665-1e96cab89577" />


📸 스크린샷: MySQL INSERT 생성 부분  

<img width="827" height="548" alt="18_link_account_sh_mysql_insert_build" src="https://github.com/user-attachments/assets/d8093f5b-ea82-4e4f-ac77-ac0dd81fe38e" />


### 6.2 Oracle 결과 파일 확인
스크립트 실행 중 생성되는 파일(`/tmp/link_account/oracle_utf8.txt`)을 열어 조회 결과를 확인했다.

📸 스크린샷: oracle_utf8.txt  

<img width="616" height="187" alt="07_oracle_export_result_file" src="https://github.com/user-attachments/assets/3616932a-9a22-425c-8ddb-8adcdcce0111" />


---

## 7. shared.sql (실제 계정 연동 로직)

`shared.sql`은 staging 테이블을 기반으로 실제 RADIUS 테이블을 갱신한다.

### 주요 흐름
- `dept_vlan`을 참조해 `groupname` 계산
- 퇴직자 삭제: staging에 없는 username 삭제
- 신규 추가: `radcheck(Cleartext-Password)` / `radusergroup(groupname)` 삽입
- 변경 반영: groupname / password 갱신

📸 스크린샷: shared.sql 전체  

<img width="752" height="588" alt="14_shared_sql_full" src="https://github.com/user-attachments/assets/1092725d-38e6-4b4a-9312-aad7ee077c95" />


📸 스크린샷: radusergroup UPDATE JOIN  

<img width="490" height="131" alt="15_update_radusergroup_join" src="https://github.com/user-attachments/assets/111626b5-2f41-415d-aba1-f2da808f8464" />


---

## 8. 결과 검증 (MySQL)

### 8.1 staging(temp_empolyee) 반영 확인
📸 스크린샷 (staging 조회 예시)  

<img width="976" height="452" alt="10_mysql_temp_employee_select" src="https://github.com/user-attachments/assets/cf206ec7-cf9c-405f-8b30-f5d7720ff2fa" />


### 8.2 radcheck 반영 확인
📸 스크린샷  

<img width="592" height="269" alt="12_mysql_radcheck_select" src="https://github.com/user-attachments/assets/ee953e5c-e03a-466b-bfb9-5fe1ef276c8e" />


### 8.3 radusergroup 반영 확인
📸 스크린샷  

<img width="500" height="259" alt="13_mysql_radusergroup_select" src="https://github.com/user-attachments/assets/866b45da-1ddf-4cd8-aea2-d35b9241cd56" />

### 8.2 dept_vlan 
📸 스크린샷

<img width="515" height="270" alt="11_mysql_dept_vlan_select" src="https://github.com/user-attachments/assets/e4d871f4-f5f7-4a7a-bcd7-ff40314341b3" />

---

## 9 자동화 (cron)

`/etc/crontab`에 등록하여 주기 실행하도록 구성했다.

예시:
```cron
# Account Linking Update SH
1 * * * * root /bin/sh /root/radius/link_account.sh >> /var/log/link_account.log 2>&1
```

📸 스크린샷  

<img width="966" height="626" alt="19_crontab_schedule" src="https://github.com/user-attachments/assets/77ce2760-4593-4a9c-b6ed-78f1d2b1fc58" />

