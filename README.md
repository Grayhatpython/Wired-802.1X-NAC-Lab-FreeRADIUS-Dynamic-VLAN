# 802.1X 기반 네트워크 접근통제(NAC) 구축 (FreeRADIUS)

본 저장소는 **기업(Enterprise) 환경을 가정한 802.1X NAC 랩**입니다.  
**Access Switch(Authenticator) + FreeRADIUS(Authentication Server)** 를 통해 단말의 네트워크 접속을 인증하고, 정책에 따라 **포트 제어 / 동적 VLAN 할당 / DHCP 기반 IP 부여 / 로그(Accounting·Syslog) 수집·가공**까지 단계적으로 구현합니다.

> 진행 중(Working In Progress): 설계/구현/문서화를 동시에 진행합니다.  
> README는 “설계(Design) → 구현(Implementation) → 검증(Verification) → 자동화(Automation)” 순으로 지속 업데이트됩니다.

> 일부 스크린샷/토폴로지/설정 값은 현재 구성과 차이가 있을 수 있으며, 최신 상태는 **레포지토리의 최근 커밋**을 기준으로 합니다.

---

## 1) 전체 구성도

### 🧱 가상화 환경(VMware + EVE-NG) 구성

<img width="300" height="310" alt="스크린샷 2026-01-22 094737" src="https://github.com/user-attachments/assets/855a9778-679e-48e2-9227-1802b60cd33a" />


### 🧱EVE NG Lab
<img width="1239" height="986" alt="스크린샷 2026-02-03 164404" src="https://github.com/user-attachments/assets/5220bc87-f474-4469-96b3-142dc6b3585d" />


### 🧱 VLAN & Subnet Design (Enterprise)

<img width="1771" height="988" alt="스크린샷 2026-02-12 174246" src="https://github.com/user-attachments/assets/d41509e0-16f9-4007-9b81-91cd08853354" />



---

## 2) VLAN / Subnet 요약

### 2.1 관리/보안 인프라
| VLAN | Name | 목적 | Subnet | Gateway(SVI) |
|---:|---|---|---|---|
| 100 | Management_Security | 보안/인증 인프라(FreeRADIUS/DHCP/Log) | 10.0.10.0/28 | 10.0.10.1 |
| 110 | Management_Device | 네트워크 장비 관리(SSH/SNMP/Syslog 등) | 10.0.11.0/28 | 10.0.11.1 |

### 2.2 서버 존
| VLAN | Name | 목적 | Subnet | Gateway(SVI) |
|---:|---|---|---|---|
| 200 | Server_Development | 개발 서버 존 | 10.0.20.0/28 | 10.0.20.1 |
| 210 | Server_Business | 행정/업무 서버 존 | 10.0.21.0/28 | 10.0.21.1 |

### 2.3 사용자 존(부서/역할 기반)
| VLAN | Name | 대상 | Subnet | Gateway(SVI) |
|---:|---|---|---|---|
| 300 | User_Security | 보안 부서 | 10.0.30.0/24 | 10.0.30.1 |
| 310 | User_Development | 개발 부서 | 10.0.31.0/24 | 10.0.31.1 |
| 320 | User_Business | 행정/업무 부서 | 10.0.32.0/24 | 10.0.32.1 |
| 330 | User_Partner_Company | 협력업체 | 10.0.33.0/24 | 10.0.33.1 |
| 340 | User_Guest | 방문자(게스트) | 10.0.34.0/24 | 10.0.34.1 |

### 2.4 직원 생활망(사내망과 분리)
| VLAN | Name | 목적 | Subnet | Gateway(SVI) |
|---:|---|---|---|---|
| 500 | User_Coporation_Life | **직원 생활망(인터넷 전용, 사내망 접근 제한)** | 172.16.50.0/24 | 172.16.50.1 |

### 2.5 802.1X 인증 흐름 VLAN
| VLAN | Name | 목적 | Subnet | Gateway(SVI) |
|---:|---|---|---|---|
| 800 | Auth_Fail | 인증 실패(격리/제한) | 10.0.80.0/24 | 10.0.80.1 |
| 810 | Auth_Service | 인증/온보딩(임시) | 10.0.81.0/24 | 10.0.81.1 |

---

## 3) 구성 요소와 역할

- **Supplicant(요청자)**: 인증 대상 단말(Windows 등)
  - 링크 업 시 EAPOL로 802.1X 인증 시작
- **Authenticator(인증자)**: Access Switch
  - EAPOL을 중계하고, RADIUS 결과에 따라 포트 제어 및 VLAN 전환
- **Authentication Server(인증 서버)**: FreeRADIUS
  - 인증(PEAP/EAP-TLS 등) 수행
  - 정책 기반 VLAN ID(Tunnel-Private-Group-ID) 반환 가능
- **DHCP Server**
  - VLAN별 주소 풀 운영 및 IP 할당
  - 인증 성공 후 해당 VLAN에서 정상 IP 할당이 되는지 검증
- **Syslog → MySQL(DB)**
  - DHCP 이벤트를 포함한 시스템 로그를 DB에 적재
  - 이후 **DHCPACK 로그 파싱 → 별도 테이블로 정규화**하여 활용
- **Distribution/Core(게이트웨이)**
  - SVI 기반 Inter-VLAN Routing
  - 상단 Edge/Internet 구간과 연동

---

## 4) 접근통제(Policy) 개요

### 4.1 보안/인증 인프라(VLAN100)
- 관리자만 접근 (기본 차단 + 예외 허용)
- 서비스 포트 단위로 최소 허용
  - 예: SSH(22), RADIUS(1812/1813), DHCP(67/68), DNS(53), HTTP/HTTPS(80/443)

### 4.2 관리 VLAN(VLAN110)
- 장비 관리 트래픽(SSH/SNMP/Syslog) 중심
- FreeRADIUS/DHCP/로그 서버와 운영 통신 허용

### 4.3 직원 생활망(VLAN500)
- **사내망(User/Server/Management)으로 라우팅 금지**
- 인터넷(외부)만 허용하는 “분리망” 컨셉

---

## 5) 단계별 구현 계획 (Roadmap)

### 6.1 네트워크 인프라(EVE-NG)
- [x] L2/L3 스위치/엣지/서버 기본 연결
- [x] VMware NAT(vmnet8) 기반 외부 인터넷 접근 경로 확보(패키지 설치/업데이트 목적)
- [x] VLAN/게이트웨이(SVI) 뼈대 구성

### 6.2 802.1X 인증 인프라(FreeRADIUS) 
→ [인증 서버 인프라 구성](./docs/인증%20서버%20인프라%20구성) ,  [클라이언트 802.1X 인증 테스트](./docs/클라이언트%20802.1X%20인증%20테스트)

### 6.4 DHCP 구성 및 Syslog→MySQL로 수집한 DHCP 로그를 DHCPACK 기준으로 파싱하고, IP 할당현황/고정IP 관리를 자동화 
→ [DHCP_MYSQL을 활용한 IP 주소 관리 자동화](./docs/DHCP_MYSQL을%20활용한%20IP%20주소%20관리%20자동화)

### 6.5 FreeRADIUS–DB 연계를 통해 사용자·단말·IP를 통합 매핑하고, 802.1X 기반 접근통제 및 IP 할당 추적 체계
→ [사용자 계정 연동 및 사용자 인증과 IP 주소 할당](./docs/사용자%20계정%20연동%20및%20사용자%20인증과%20IP%20주소%20할당)

---
