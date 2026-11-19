# PHANTOM 데이터셋 — 시나리오 2\
: 스피어 피싱에서 Windows 엔드포인트 침해까지

시나리오 2는 가짜 VPN 포털 피싱으로 시작해 WinRM으로 Windows 클라이언트(SEC-CLIENT-01)에서 자격증명 탈취·지속성·수집·유출까지 수행한 엔드포인트 공격 데이터셋이다.

## 1. 개요

시나리오 2는 2026-11-19 05:43:24Z(KST 14:43)에 시작해 약 5분 50초간 실행된 9단계 엔드포인트 공격 체인 시뮬레이션이며, 가짜 VPN 포털 피싱부터 Windows 클라이언트(SEC-CLIENT-01) 접근, 자격증명 접근, 지속성, 탐색, 횡적 이동, 수집, 유출까지를 다룬다. 실행 로그는 422줄이고 결과는 17 OK / 4 WARN / 0 FAIL이다. 로그는 SIEM 사본이 아니라 각 보안 장비에서 직접 읽었고, 값은 재작성하지 않았다. 수집 구간은 이벤트 타임스탬프 기준으로 시나리오 시작~종료(05:43:24Z–05:49:19Z)만 남겼다.

| 항목 | 내용 |
| --- | --- |
| 데이터셋 ID | ocsf\_scenario2\_phish\_20261119T054324Z |
| 수행 기관 | CSS LAB (KENTECH) |
| 수집일 | 2026-11-19 |
| 실행 구간 | 05:43:24Z – 05:49:19Z (마지막 기록 05:49:16Z) |
| 시나리오 유형 | 피싱, 유효 계정 사용, Windows 원격 관리(WinRM) 접근, 자격증명 접근, 지속성, 수집, 유출 |
| 단계 수 | 9단계 (STEP 1–5 + 3\_EDR, 3\_PERSIST, 3\_ENUM, 3\_LATERAL) |
| 실행 결과 | 17 OK / 4 WARN / 0 FAIL |
| 수집 소스 | 20개 (7개 센서 패밀리 + CrowdStrike Falcon) |
| 총 용량 | 약 810 KB |

## 2. 수집 환경

공격자 VM이 가짜 VPN 포털을 운영하고, Windows 클라이언트(SEC-CLIENT-01)는 관리망(192.168.121.0/24)의 WinRM(:5985)으로 접근한다. 내부 클라이언트(10.10.20.20)로의 SSH 접근은 시간 초과되어 WinRM 경로로 대체되었다.

| 호스트 | IP | 역할 |
| --- | --- | --- |
| 공격자 VM (atk-kali-01) | 10.10.1.100 (WAN) / 10.10.50.80 (관리) | PHANTOM 실행, 가짜 VPN 포털, 유출 수신 |
| sec-waf-01 | 10.10.10.40 | VPN 인증 시도 대상 |
| 내부 클라이언트 | 10.10.20.20 | SSH 접근 시도 대상(시간 초과) |
| SEC-CLIENT-01 | 192.168.121.46 (관리망) | Windows 클라이언트, WinRM(:5985) 접근 대상 |
| sec-decoy-01 | 10.10.40.20 | Decoy 서버 (finance 공유) |
| sec-fw-01 | - | nftables 방화벽 |
| sec-ids-dmz-01 / ips / int-server / int-user | - | Suricata IDS |
| sec-wazuh-01 | 10.10.50.42 | Wazuh HIDS |
| sec-fail2ban-01 | 10.10.50.43 | Fail2ban |

SIEM은 10.10.50.90(:9200)이나 이 데이터셋은 SIEM을 거치지 않고 각 보안 장비에서 직접 수집했다. 내부 클라이언트 접근은 피싱으로 수집한 계정을 사용하도록 설정되어 있고, WinRM 대상은 실행 시점에 자동 감지된다. SMTP 서버가 기동되어 있지 않아 피싱 메일 발송은 실패했다(WARN).

## 3. 시나리오 구성

시나리오는 피싱 포털 기동부터 아카이브 유출까지 9단계로 진행된다. 시각은 UTC 기준 각 단계 시작 시각이다.

| 단계 | 시각 | 이름 | 내용 |
| --- | --- | --- | --- |
| 1 | 05:43:27 | 가짜 VPN 포털 및 피싱 메일 | 가짜 VPN 포털 기동, 피싱 메일 발송 시도(SMTP 거부) |
| 2 | 05:43:37 | VPN 인증 시도 | 수집한 계정으로 VPN/원격 서비스 인증 시도 |
| 3 | 05:43:47 | 내부 클라이언트 접근 | SSH 접근 시도(시간 초과), RDP 프로브, WinRM 자동 감지 |
| 3\_EDR | 05:44:05 | Windows EDR 트리거 | WinRM(NTLM) 연결 후 LSASS 조회, SAM 하이브 백업, Shadow Copy 조회 등 자격증명 접근 |
| 3\_PERSIST | 05:45:54 | 지속성 및 브라우저 자격증명 | 예약 작업 생성, 브라우저 자격증명 수집 단계 |
| 3\_ENUM | 05:46:19 | 도메인 탐색 | 계정·공유·서브넷 탐색 |
| 3\_LATERAL | 05:46:31 | 횡적 이동 시도 | 내부 서버로 SMB/WinRM 피벗 시도 |
| 4 | 05:46:52 | 클라이언트→Decoy 수집 | Decoy SMB 수집 시도, WinRM 파일 전송 (47,165,236 바이트 / 126개 파일) |
| 5 | 05:49:16 | 아카이브 및 유출 | finance 아카이브 생성 후 가짜 포털로 전송 (13,534,350 바이트, HTTP 302) |

## 4. 수집 장비 및 로그 소스

로그는 네트워크·호스트·엔드포인트 계층의 센서에서 수집했다. 아래 줄 수는 이번 실행 구간에 실제로 수집된 양이다.

| 패밀리 | 소스 | 호스트 | 계층 | 줄 수 |
| --- | --- | --- | --- | ---: |
| 04\_nftables\_firewall | kernel | sec-fw-01 | 네트워크 | 2,908 |
| 01\_suricata | suricata\_eve | sec-ips-01 | 네트워크 | 39 |
| 01\_suricata | suricata\_eve | sec-ids-dmz-01 | 네트워크 | 32 |
| 01\_suricata | suricata\_eve | sec-ids-int-user-01 | 네트워크 | 8 |
| 01\_suricata | suricata\_eve | sec-ids-int-server-01 | 네트워크 | 1 |
| 01\_suricata | suricata\_eve | sec-decoy-01 | 네트워크 | 1 |
| 02\_zeek | zeek\_conn | sec-ids-dmz-01 | 네트워크 | 28 |
| 02\_zeek | zeek\_ssh | sec-ids-dmz-01 | 네트워크 | 1 |
| 03\_waf\_modsecurity | linux\_audit | sec-waf-01 | 호스트 | 31 |
| 03\_waf\_modsecurity | waf\_access / waf\_modsec | sec-waf-01 | 애플리케이션 | 0 |
| 05\_wazuh | wazuh\_alerts | sec-wazuh-01 | 호스트 | 0 |
| 06\_fail2ban | fail2ban | sec-fail2ban-01 | 호스트 | 0 |
| 08\_crowdstrike\_falcon | falcon | - | 엔드포인트 | 0 |
| 02\_zeek | zeek\_dns / http / files / ssl / notice / weird | sec-ids-dmz-01 | 네트워크 | 0 |

※ PHANTOM 실행 로그(공격자 VM, 422줄)는 시나리오 정답(ground truth) 기록으로 별도 보관한다. LSASS 조회, SAM 백업, 예약 작업, 파일 전송(126개) 등 클라이언트 측 행위의 상세는 이 실행 로그에 남는다.

### 수집 관련 참고 (중요)

- **시나리오 2는 네트워크 흔적이 작은 엔드포인트 중심 체인이다.** 가짜 포털은 공격자 VM에 있고 실제 행위는 내부 클라이언트(WinRM/SMB)에서 일어나므로, DMZ 웹(WAF)·Suricata·Zeek에는 거의 남지 않는다. 연결 흔적은 방화벽 커널 로그(2,908건)에 가장 많이 남는다.
- **Windows 클라이언트(SEC-CLIENT-01)의 엔드포인트 텔레메트리는 이 데이터셋에 수집되지 않았다.** CrowdStrike Falcon(0건)·Wazuh(0건) 모두 이번 구간에 신규 경보가 없다. LSASS 조회·SAM 백업·예약 작업 등 EDR 유발 행위는 시나리오 실행 로그(ground truth)에만 기록되어 있다. 클라이언트 측 탐지 로그가 필요하면 Windows 이벤트 로그·EDR 수집을 별도로 추가해야 한다.
- **Falcon(리눅스, sec-web-01)이 0건인 것**은 이 시나리오가 웹 서버를 건드리지 않기 때문이다.
- **Zeek dns·http·ssl 등이 0건인 것**은 이 구간에 DMZ를 지나는 해당 트래픽이 없기 때문이다.
- 내부 구간(int-user/int-server)과 클라이언트에 직접 붙는 네트워크 센서가 없어 대용량 WinRM 전송(47MB)의 상세 패킷 로그는 남지 않는다. 이는 센서 배치의 한계다.

## 5. MITRE ATT&CK 매핑

| 전술 | 기법 ID | 기법 | 근거 단계 |
| --- | --- | --- | --- |
| Initial Access | T1566 | Phishing | 1 |
| Initial Access | T1078 | Valid Accounts | 2 |
| Lateral Movement | T1021.004 | Remote Services: SSH | 3 |
| Lateral Movement | T1021.006 | Remote Services: Windows Remote Management | 3\_EDR |
| Credential Access | T1003 | OS Credential Dumping (LSASS, SAM) | 3\_EDR |
| Credential Access | T1555.003 | Credentials from Web Browsers | 3\_PERSIST |
| Persistence | T1053.005 | Scheduled Task/Job: Scheduled Task | 3\_PERSIST |
| Discovery | T1087 | Account Discovery | 3\_ENUM |
| Discovery | T1135 | Network Share Discovery | 3\_ENUM |
| Discovery | T1018 | Remote System Discovery | 3\_ENUM |
| Lateral Movement | T1021.002 | Remote Services: SMB/Windows Admin Shares | 3\_LATERAL |
| Collection | T1039 | Data from Network Shared Drive | 4 |
| Collection | T1560.001 | Archive Collected Data: Archive via Utility | 5 |
| Exfiltration | T1041 | Exfiltration Over C2 Channel | 5 |
