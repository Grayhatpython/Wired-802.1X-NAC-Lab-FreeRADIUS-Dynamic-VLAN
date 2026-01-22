# 🧪 802.1X 실습 기록 (FreeRADIUS + DHCP + VLAN 810 + Windows Client)

> 목표: **DBMS 기반으로 IP 사용 현황/임대 이력/고정 예약을 통합 관리하고, DHCP 설정 변경·배포를 자동화하여 주소 충돌 및 수동 작업을 최소화한다

## 1) DHCP 개념 정리 + 환경설정 기록

이번 실습의 DHCP는 단순히 “IP를 준다”가 아니라,
- VLAN이 여러 개일 때 DHCP 풀/서브넷을 어떻게 나누고
- 임대(lease)가 어떻게 갱신되고
- 고정 IP 예약은 어떻게 넣는지
까지 같이 정리했습니다.

---

### 1-1. DHCP 주소 할당 절차(DORA)

<img width="686" height="533" alt="dhcp_04" src="https://github.com/user-attachments/assets/016b00a1-bc0b-4b37-b0ef-52cf3b0b1c92" />


- **D**iscover: 클라이언트가 “DHCP 서버 있나요?” 브로드캐스트
- **O**ffer: 서버가 “이 IP 줄게요(제안)”
- **R**equest: 클라이언트가 “그 IP로 주세요(요청)”
- **A**ck: 서버가 “확정(임대 시작)” → 이때부터 클라이언트는 IP 사용

---

### 1-2. 임대(Lease) 갱신 절차(T1/T2)

<img width="941" height="900" alt="dhcp_05" src="https://github.com/user-attachments/assets/3d0a2848-1637-4ead-9f55-fa50f93ad0a2" />


- **T1(기본 50%)**: 임대 시간의 절반 시점에서 **기존 DHCP 서버로 유니캐스트 갱신** 시도
- **T2(기본 87.5% = 7/8)**: 갱신이 안 되면 **브로드캐스트로 재바인딩(Rebind)** 시도
- 만약 갱신 과정에서 서버로부터 **DHCPNAK**를 받으면:
  - 클라이언트는 그 IP를 더 이상 쓰면 안 되고
  - 보통 즉시 DHCP 초기 상태로 돌아가서(DISCOVER부터) 재할당을 시도합니다.

> 정리: DHCPACK(승인)와 DHCPNAK(거절)는 완전히 다릅니다.
> - DHCPACK: 임대 유지/갱신 성공
> - DHCPNAK: 현재 IP는 유효하지 않음 → 즉시 재할당 프로세스

---

### 1-3. 임대 이력(lease) 관리 파일 확인

ISC DHCP는 임대 기록을 보통 아래 파일에 남깁니다.
- `/var/lib/dhcp/dhcpd.leases`

<img width="684" height="523" alt="dhcp_01" src="https://github.com/user-attachments/assets/31e5b5b7-5dca-4e89-93f2-c9dc3c9c2a53" />


이 파일에서 자주 보는 항목:
- `lease <IP> { ... }`
- `starts / ends` : 임대 시작/만료 시각
- `binding state active` : 현재 활성 임대
- `hardware ethernet ...` : 임대에 매칭된 MAC

---

### 1-4. 고정 IP(예약) 할당: `host { fixed-address }`

- DHCP “풀(range)”로 주는 대신, 특정 MAC에 특정 IP를 **예약**할 수 있습니다.

<img width="927" height="711" alt="dhcp_03" src="https://github.com/user-attachments/assets/6f9f8498-12be-4401-9630-09dff2317015" />


동작 포인트:
- DHCP 서버를 재시작해도 `host` 항목이 살아있으면,
  - 해당 MAC이 요청할 때는 **pool이 아니라 fixed-address가 우선** 적용됩니다.

---

### 1-5. VLAN별 DHCP 풀 분리: 서버가 어떤 기준으로 풀을 고르나?

이번 실습은 서버가 VLAN 서브인터페이스에 직접 붙는 형태로 구성했고,
- DHCP 서버는 `ens4.810`, `ens4.300`, `ens4.310` ... 처럼 **VLAN별 인터페이스**에서 요청을 수신합니다.
- 따라서 “들어온 인터페이스”에 대응하는 `subnet {}` 블록을 타고, 그 안의 `range`에서 Offer를 만듭니다.

---

### 1-6. Netplan: VLAN 서브인터페이스 여러 개 구성 중 발생한 에러

#### 증상
- VLAN 인터페이스(예: `ens4.810`, `ens4.300` 등)를 여러 개 정의하고 `netplan apply` 하면 assertion 에러 발생.

<img width="919" height="135" alt="dhcp_06" src="https://github.com/user-attachments/assets/99101d1b-c6f0-46f7-8db0-1ea0ee8d84c0" />


#### 원인
- VLAN 인터페이스 정의에는 `link: ens4` 같이 **부모 링크(Parent NIC)** 정보가 필요합니다.
- NetworkManager 렌더러 사용 시, 부모 인터페이스가 netplan에 전혀 선언되어 있지 않으면 VLAN 생성 과정에서 실패할 수 있습니다.

#### 해결(핵심)
- `ens4`를 “주소 없이”라도 선언(예: `dhcp4: no`)해두면 VLAN 인터페이스가 정상 생성됨.

VLAN 인터페이스가 정상적으로 올라온 상태는 아래에서 확인.

<img width="865" height="115" alt="dhcp_07" src="https://github.com/user-attachments/assets/4c056216-9fdb-4cd3-b71f-da3683c8890a" />


현재 적용된 netplan 구성은 `netplan get`으로 빠르게 확인할 수 있습니다.

<img width="605" height="592" alt="dhcp_08" src="https://github.com/user-attachments/assets/1c4d9820-5674-45aa-9a50-c84213ff117d" />


VLAN 서브인터페이스 정의 스냅샷(예시):

<img width="311" height="505" alt="dhcp_09" src="https://github.com/user-attachments/assets/53eb210a-93c4-42fb-9325-564640539639" />

<img width="547" height="526" alt="dhcp_10" src="https://github.com/user-attachments/assets/e3f2be8a-dd83-405c-afaa-7819c54969a2" />

<img width="531" height="523" alt="dhcp_11" src="https://github.com/user-attachments/assets/96ccd244-d1e9-43e1-84b0-50b01c9424ac" />

<img width="515" height="283" alt="dhcp_12" src="https://github.com/user-attachments/assets/b920088d-646b-4a59-867f-4f60ac67ad55" />

---

### 1-7. ISC DHCP 서버 설정

#### (1) DHCP 서버가 리슨할 인터페이스 지정
- `/etc/default/isc-dhcp-server`

<img width="926" height="701" alt="dhcp_14" src="https://github.com/user-attachments/assets/8819b884-bf88-427a-96e2-8778dab13c60" />


예시:
```ini
INTERFACESv4="ens4.810 ens4.300 ens4.310 ens4.320 ens4.330 ens4.340"
```

#### (2) VLAN별 subnet / range 구성
- `/etc/dhcp/dhcpd.conf`

<img width="676" height="648" alt="dhcp_15" src="https://github.com/user-attachments/assets/a1ca7618-b71e-42f8-acde-d957fd9111bd" />

<img width="632" height="601" alt="dhcp_16" src="https://github.com/user-attachments/assets/46df02b8-d0c7-4376-a7df-7ebf75626330" />


요약(스크린샷 기준)
- VLAN 300: `10.0.30.0/24`, GW `10.0.30.1`, range `10.0.30.15~10.0.30.250`
- VLAN 310: `10.0.31.0/24`, GW `10.0.31.1`, range `10.0.31.15~10.0.31.250`
- VLAN 320: `10.0.32.0/24`, GW `10.0.32.1`, range `10.0.32.15~10.0.32.250`
- VLAN 330: `10.0.33.0/24`, GW `10.0.33.1`, range `10.0.33.15~10.0.33.250`
- VLAN 340: `10.0.34.0/24`, GW `10.0.34.1`, range `10.0.34.15~10.0.34.250`

> 참고: 각 subnet에 `default-lease-time 1800(30분)`, `max-lease-time 3600(60분)`으로 설정되어 있었습니다.

#### (3) 서비스 재시작 및 프로세스 확인

<img width="796" height="20" alt="dhcp_17" src="https://github.com/user-attachments/assets/b439334b-bc41-45ef-bda3-d5ad0469f9a4" />


```bash
sudo systemctl restart isc-dhcp-server
```

프로세스가 실제로 어떤 인터페이스로 붙어서 떠 있는지 확인:

<img width="918" height="152" alt="dhcp_18" src="https://github.com/user-attachments/assets/a239c304-eacf-4477-b515-435612e1c684" />


---

### 1-8. VLAN 서브인터페이스 통신 검증

- 서버는 여러 VLAN 서브인터페이스를 갖고 있고,
- DSW에서 각 VLAN 대역의 서버 IP로 ping이 성공한 것을 확인.

<img width="1659" height="680" alt="dhcp_13" src="https://github.com/user-attachments/assets/0286aadb-b608-4501-8018-604cedc4bd40" />


---

