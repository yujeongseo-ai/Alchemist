# PHANTOM 데이터셋 — 통합 시나리오(Full Chain): 7-Phase 엔드투엔드 캠페인

통합 시나리오는 시나리오 1\~3의 공격·방어 요소를 하나의 7-Phase 캠페인(정찰 → 유출)으로 묶어 실행한 데이터셋이다.

## 1. 개요

통합 시나리오는 2026-11-16 10:10:32Z(KST 19:10)에 시작해 약 11분 49초간 실행된 7-Phase / 17-Step 캠페인이며, OCSF 스키마 매핑을 위해 각 보안 센서에서 로그를 수집했다. 실행 로그는 1,793줄이고 결과는 29 OK / 21 WARN / 0 FAIL이다. 로그는 SIEM 사본이 아니라 각 보안 장비에서 직접 읽었고, 값은 재작성하지 않았다. 수집 구간은 이벤트 타임스탬프 기준으로 시나리오 시작~종료(10:10:32Z–10:22:21Z)만 남겼다. 이 실행은 Zeek 체크섬 오프로드 보정(`ignore_checksums`) 적용 후 수집한 것으로, Zeek http·files·ssh가 정상 기록되었다.

| 항목 | 내용 |
| --- | --- |
| 데이터셋 ID | ocsf\_full\_chain\_20261116T101032Z |
| 수행 기관 | CSS LAB (KENTECH) |
| 수집일 | 2026-11-16 |
| 실행 구간 | 10:10:32Z – 10:22:21Z (약 11분 49초) |
| 시나리오 유형 | 정찰, 소셜 엔지니어링, 초기 접근, 웹 공격, 횡적 이동, 지속성·방어 회피, 유출 |
| 단계 수 | 7 Phase / 17 Step |
| 실행 결과 | 29 OK / 21 WARN / 0 FAIL |
| 수집 소스 | 20개 (7개 센서 패밀리 + CrowdStrike Falcon) |
| 총 용량 | 약 77 MB |

## 2. 수집 환경

통합 시나리오는 시나리오 1\~3에서 사용한 호스트를 한 번에 사용한다.

| 호스트 | IP | 역할 |
| --- | --- | --- |
| 공격자 VM (atk-kali-01) | 10.10.1.100 (WAN) / 10.10.50.80 (관리) | 캠페인 실행, 피싱 포털(:8443), 유출 수신 |
| sec-waf-01 | 10.10.10.40 | WAF (nginx + ModSecurity CRS) |
| sec-web-01 | 10.10.10.20 | 웹 서버 (SQL 인젝션·웹셸 대상, Linux Falcon) |
| sec-db-01 | 10.10.21.20 | DB 서버 (MySQL) |
| sec-decoy-01 | 10.10.40.20 | Decoy 서버 (finance 공유, SMB 수집 대상) |
| 내부 클라이언트 | 10.10.20.20 | SSH 접근 시도 대상 |
| SEC-CLIENT-01 | 192.168.121.46 (관리망) | Windows 클라이언트 (WinRM/NTLM, 클라이언트 Falcon) |
| sec-fail2ban-01 | 10.10.50.43 | Fail2ban |
| sec-wazuh-01 | 10.10.50.42 | Wazuh HIDS |
| sec-wireguard-01 | 10.10.50.44 | WireGuard VPN (C2 채널) |
| sec-fw-01 | - | nftables 방화벽 |
| sec-ids-dmz-01 / ips / int-server / int-user / decoy | - | Suricata IDS |
| SIEM | 10.10.50.90 | 로그 수집 (:9200) — 이 데이터셋은 SIEM을 거치지 않고 장비 직접 수집 |

EDR은 Linux(sec-web-01)와 Windows 클라이언트(SEC-CLIENT-01) 모두 CrowdStrike Falcon을 사용한다.

## 3. 시나리오 구성

캠페인은 정찰부터 유출까지 7 Phase로 구성된다. 시각은 UTC 기준 각 Phase 시작 시각이다.

| Phase | 이름 | 시각 | 내용 |
| --- | --- | --- | --- |
| P1 | 정찰 | 10:10:46 | 인프라 포트 스캔, 웹 핑거프린트 |
| P2 | 소셜 엔지니어링 | 10:11:02 | 피싱 포털 배포, SSH 인증 실패로 Fail2ban 트리거 |
| P3 | 초기 접근 | 10:12:44 | WinRM 엔드포인트 확인, 피싱 계정 검증, WireGuard C2 사이클 1·2 |
| P4 | 웹 공격 | 10:15:34 | SQL 인젝션, 웹셸 업로드·RCE 확인, Linux EDR 아티팩트 |
| P5 | 횡적 이동 및 수집 | 10:16:14 | DB 서버 이동·덤프, SMB 수집, Wazuh FIM 3이벤트, WireGuard 사이클 3 |
| P6 | 지속성 및 방어 회피 | 10:18:17 | 지속성 설치, 로그 정리, Linux EDR 아티팩트, WireGuard 사이클 4·5 |
| P7 | 유출 | 10:18:51 | 스테이징 파일 수집, 아카이브 생성, C2 비콘, 다채널 반출 |

전체 17개 Step은 다음 순서다.

1. Infrastructure port scan & web fingerprint
2. SSH spray noise — Fail2ban trigger
3. Phishing portal deployment & spear-phishing email
4. Phished credential validation vs VPN/remote service
5. Internal client access via phished VPN credentials
6. WireGuard C2 채널 개설 — 사이클 1+2
7. SQL injection probe & dump
8. Webshell upload & RCE confirmation
9. SSH pivot: web server → DB server
10. Database dump & in-memory staging
11. Triangle lateral movement: DB → Decoy / DB → Client
12. SMB collection from decoy share
13. Wazuh FIM trigger — 3-event syscheck cycle
14. WireGuard C2 사이클 3/5 — 중량 데이터 스테이징
15. Pull staged files & build exfil archive
16. C2 beaconing: HTTP, DNS, ICMP (all hosts)
17. Multi-channel exfiltration: cloud PUT, SCP, SSL, SMTP, relay

## 4. 수집 장비 및 로그 소스

로그는 네트워크·애플리케이션·호스트·엔드포인트 계층의 센서에서 수집했다. 아래 줄 수는 이번 실행 구간에 실제로 수집된 양이다.

| 패밀리 | 소스 | 호스트 | 줄 수 |
| --- | --- | --- | ---: |
| 01\_suricata | suricata\_eve | sec-ids-dmz-01 | 27,598 |
| 01\_suricata | suricata\_eve | sec-ips-01 | 24,334 |
| 01\_suricata | suricata\_eve | sec-ids-int-server-01 | 3,368 |
| 01\_suricata | suricata\_eve | sec-ids-int-user-01 | 32 |
| 01\_suricata | suricata\_eve | sec-decoy-01 | 52 |
| 02\_zeek | zeek\_conn | sec-ids-dmz-01 | 7,126 |
| 02\_zeek | zeek\_files | sec-ids-dmz-01 | 7,927 |
| 02\_zeek | zeek\_http | sec-ids-dmz-01 | 7,924 |
| 02\_zeek | zeek\_ssh | sec-ids-dmz-01 | 64 |
| 02\_zeek | zeek\_weird | sec-ids-dmz-01 | 42 |
| 02\_zeek | zeek\_notice | sec-ids-dmz-01 | 2 |
| 02\_zeek | zeek\_dns / ssl | sec-ids-dmz-01 | 0 |
| 03\_waf\_modsecurity | waf\_modsec | sec-waf-01 | 40,950 |
| 03\_waf\_modsecurity | waf\_access | sec-waf-01 | 7,924 |
| 03\_waf\_modsecurity | linux\_audit | sec-waf-01 | 1,835 |
| 04\_nftables\_firewall | kernel | sec-fw-01 | 18,391 |
| 05\_wazuh | wazuh\_alerts | sec-wazuh-01 | 158 |
| 06\_fail2ban | fail2ban | sec-fail2ban-01 | 9 |
| 08\_crowdstrike\_falcon | falcon | sec-web-01 / SEC-CLIENT-01 | 9 |

※ PHANTOM 실행 로그(공격자 VM, 1,793줄)는 시나리오 정답(ground truth) 기록으로 별도 보관한다. WireGuard C2 핸드셰이크(5회 사이클)는 OCSF 매핑 대상이 아니라 데이터셋에서 제외했으며, 실행 로그에만 남는다.

### 수집 관련 참고

- **단일 실행으로 세 시나리오의 탐지가 모두 나타난다.** 웹 공격(Suricata·WAF·Zeek), 방어 어플라이언스(Wazuh·Fail2ban), 엔드포인트(Falcon)가 한 데이터셋에 함께 들어 있다.
- **Falcon 9건에 Windows 클라이언트 탐지 1건(MaliciousPowershell, SEC-CLIENT-01)이 포함된다.** 나머지 8건은 sec-web-01(Linux)의 웹셸·리버스셸·자격증명 접근 탐지다. 탐지(event) 시각 기준으로 구간만 포함했다.
- **Zeek dns·ssl은 0건**으로, 이 구간에 DMZ를 지나는 DNS·TLS 트래픽이 없기 때문이다.
- **WireGuard는 OCSF 매핑 클래스가 없어 제외**했다(ocsf\_mapping.csv 기준).

### 대표 탐지 규칙 (이번 실행 실측)

**ModSecurity CRS (sec-waf-01, 상위)**: 920350 Host header is numeric IP(7,116), 949110 Inbound Anomaly Score Exceeded(7,021), 913100 Scanner User-Agent(3,829), 959100 Outbound Anomaly Score(3,234), 951230 MySQL Information Leakage(3,233), 942100 SQLi via libinjection(3,080).

**Suricata (ET, dmz·ips·int-server 합산, 상위)**: 2016673 SQL Errors in HTTP 500(6,465), 2016935 SQLi Sleep Time Delay(923), 2017808 MySQL SQLi Information Schema(688), 2011041 MYSQL Benchmark in URI(604), 2009358 Nmap NSE User-Agent(18).

**Wazuh (sec-wazuh-01, 총 158건, 상위)**

| 규칙 ID | 레벨 | 설명 | 건수 |
| --- | ---: | --- | ---: |
| 5501 / 5502 | 3 | PAM 세션 열림/닫힘 | 34 / 34 |
| 5710 | 5 | sshd: 존재하지 않는 계정 로그인 시도 | 28 |
| 5402 | 3 | Successful sudo to ROOT | 20 |
| 5503 | 5 | PAM: User login failed | 17 |
| 5715 | 3 | sshd: authentication success | 11 |
| 5760 | 5 | sshd: authentication failed | 10 |
| 5551 | 10 | PAM: 단시간 다중 로그인 실패 | 2 |
| 5712 / 5763 | 10 | sshd: 브루트포스 접근 시도 | 1 / 1 |

**Fail2ban (sec-fail2ban-01, 총 9줄)**: `[sshd]` jail Found 6회, Ban 2회, Unban 1회.

**CrowdStrike Falcon (총 9건)**: BashReverseShell(T1219, 4, sec-web-01), LinWebshell(T1505.003, 2), CredentialAccessLin(T1003, 1), CredentialTheftLin(T1552, 1), **MaliciousPowershell(T1059.001, 1, SEC-CLIENT-01)**.

## 5. MITRE ATT&CK 매핑

| Phase | 기법 |
| --- | --- |
| P1 정찰 | T1046 Network Service Discovery, T1595 Active Scanning |
| P2 소셜 엔지니어링 | T1110.001 Password Guessing, T1021.004 SSH, T1566.002 Spearphishing Link |
| P3 초기 접근 | T1078 Valid Accounts, T1021.006 WinRM, T1059.001 PowerShell, T1572 Protocol Tunneling (WireGuard C2) |
| P4 웹 공격 | T1190 Exploit Public-Facing Application, T1059.004 Unix Shell, T1003.007 Proc Filesystem, T1505.003 Web Shell |
| P5 횡적 이동 및 수집 | T1021.004 SSH, T1021.002 SMB/Windows Admin Shares, T1005 Data from Local System, T1039 Data from Network Shared Drive |
| P6 지속성 및 방어 회피 | T1505.003 Web Shell, T1053.005 Scheduled Task, T1070 Indicator Removal, T1098.004 SSH Authorized Keys |
| P7 유출 | T1041 Exfiltration Over C2 Channel, T1048 Exfiltration Over Alternative Protocol, T1071 Application Layer Protocol, T1095 Non-Application Layer Protocol |
