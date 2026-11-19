# PHANTOM 데이터셋 — 시나리오 1\
: 웹 애플리케이션 침해에서 다채널 유출까지

시나리오 1은 DMZ 웹 서버 대상 SQLi → RCE → 횡적 이동 → C2 비콘 → 다채널 유출을 재현하고 보안 센서의 로그를 수집한 웹 APT 데이터셋이다.

## 1. 개요

시나리오 1은 2026-11-19 05:25:21Z(KST 14:25)에 시작해 약 4분 30초간 실행된 8단계 웹 공격 체인 시뮬레이션이며, OCSF 스키마 매핑을 위해 각 보안 센서에서 로그를 수집했다. 실행 로그는 1,276줄이고 결과는 19 OK / 13 WARN / 0 FAIL이다. 로그는 SIEM 사본이 아니라 각 보안 장비에서 직접 읽었고, 값은 재작성하지 않았다. 수집 구간은 이벤트 타임스탬프 기준으로 시나리오 시작~종료(05:25:21Z–05:29:57Z)만 남겼다.

| 항목 | 내용 |
| --- | --- |
| 데이터셋 ID | ocsf\_scenario1\_web\_20261119T052521Z |
| 수행 기관 | CSS LAB (KENTECH) |
| 수집일 | 2026-11-19 |
| 실행 구간 | 05:25:21Z – 05:29:57Z (마지막 기록 05:29:56Z) |
| 시나리오 유형 | 웹 애플리케이션 침해, 내부 이동, C2 통신, 데이터 유출 |
| 단계 수 | 8단계 |
| 실행 결과 | 19 OK / 13 WARN / 0 FAIL |
| 수집 소스 | 20개 (7개 센서 패밀리 + CrowdStrike Falcon) |
| 총 용량 | 약 76 MB |

## 2. 수집 환경

공격자 VM은 WAF(10.10.10.40)를 거쳐 DMZ 웹 서버(10.10.10.20)를 대상으로 하고, 이후 DB 서버(10.10.21.20)로 이동하는 구성이다. 웹 서버 액세스 로그의 출발지가 WAF IP로 남는 것은 이 프록시 경로 때문이다.

| 호스트 | IP | 역할 |
| --- | --- | --- |
| 공격자 VM (atk-kali-01) | 10.10.1.100 (WAN) / 10.10.50.80 (관리) | PHANTOM 시나리오 실행, 수신 서버(:8443) |
| sec-waf-01 | 10.10.10.40 | WAF (nginx + ModSecurity v3, CRS) |
| sec-web-01 | 10.10.10.20 | 웹 서버 (nginx, PHP 로그인 앱) |
| sec-db-01 | 10.10.21.20 | MySQL DB 서버 |
| sec-ids-dmz-01 | - | Suricata (DMZ) |
| sec-ips-01 | - | Suricata (IPS) |
| sec-ids-int-server-01 | - | Suricata (내부 서버존) |
| sec-fw-01 | - | nftables 방화벽 |
| sec-wazuh-01 | 10.10.50.42 | Wazuh HIDS |
| sec-fail2ban-01 | 10.10.50.43 | Fail2ban |
| sec-decoy-01 | 10.10.40.20 | Decoy 서버 (finance 공유) |

로그를 모으는 SIEM은 10.10.50.90(:9200)이나, 이 데이터셋은 SIEM을 거치지 않고 각 보안 장비에서 직접 수집했다. 호스트 방화벽(nftables) 로그는 방화벽 VM에서 커널 로그로 수집한다.

## 3. 시나리오 구성

시나리오는 정찰부터 유출, EDR 부하 생성까지 8단계로 진행된다. 시각은 UTC 기준 각 단계 시작 시각이다.

| 단계 | 시각 | 이름 | 내용 |
| --- | --- | --- | --- |
| 1 | 05:25:25 | 정찰 및 HTTP 핑거프린트 | WAF·웹 서버 포트 스캔, 디렉터리 탐색 |
| 2 | 05:25:43 | SQL 인젝션 탐색 및 자격증명 추출 | 다중 인젝션 패턴(UNION, 블라인드, 에러 기반)과 자동화 도구 사용 |
| 3 | 05:26:08 | 웹 발판·RCE·자격증명 수집 | 업로드 엔드포인트를 통한 웹셸 업로드, www-data 권한 RCE, 자격증명 수집 |
| 4 | 05:26:24 | 내부 이동 및 수집 준비 | 웹 서버 → DB 서버 SSH 이동, DB 덤프와 SMB finance 공유 접근, decoy 공격 |
| 5 | 05:27:40 | 스테이징 파일 수집·아카이브 | 수집 파일 취합 및 아카이브 생성 |
| 6 | 05:27:40 | C2 비콘 | HTTP, DNS, ICMP 대상 비콘 (웹 서버·DB 서버 발신) |
| 7 | 05:29:02 | 다채널 유출 | cloud PUT, SCP, SSL 터널, SMTP relay 등 채널별 유출 시도 |
| 8 | 05:29:49 | Falcon EDR 부하 생성 | 웹·DB 서버에서 EDR 탐지 유발 행위 집중 실행 |

## 4. 수집 장비 및 로그 소스

로그는 네트워크, 애플리케이션, 호스트, 데이터베이스, 엔드포인트 계층의 센서에서 수집했다. 아래 줄 수는 이번 실행 구간에 실제로 수집된 양이다.

| 패밀리 | 소스 | 호스트 | 계층 | 줄 수 |
| --- | --- | --- | --- | ---: |
| 01\_suricata | suricata\_eve | sec-ids-dmz-01 | 네트워크 | 26,126 |
| 01\_suricata | suricata\_eve | sec-ips-01 | 네트워크 | 23,601 |
| 01\_suricata | suricata\_eve | sec-ids-int-server-01 | 네트워크 | 2,608 |
| 01\_suricata | suricata\_eve | sec-ids-int-user-01 | 네트워크 | 11 |
| 01\_suricata | suricata\_eve | sec-decoy-01 | 네트워크 | 68 |
| 02\_zeek | zeek\_conn | sec-ids-dmz-01 | 네트워크 | 7,181 |
| 02\_zeek | zeek\_http | sec-ids-dmz-01 | 네트워크 | 7,965 |
| 02\_zeek | zeek\_files | sec-ids-dmz-01 | 네트워크 | 7,948 |
| 02\_zeek | zeek\_ssh | sec-ids-dmz-01 | 네트워크 | 117 |
| 02\_zeek | zeek\_weird | sec-ids-dmz-01 | 네트워크 | 3 |
| 02\_zeek | zeek\_dns / ssl / notice | sec-ids-dmz-01 | 네트워크 | 0 |
| 03\_waf\_modsecurity | waf\_modsec | sec-waf-01 | 네트워크/애플리케이션 | 44,498 |
| 03\_waf\_modsecurity | waf\_access | sec-waf-01 | 애플리케이션 | 7,965 |
| 03\_waf\_modsecurity | linux\_audit | sec-waf-01 | 호스트 | 6 |
| 04\_nftables\_firewall | kernel | sec-fw-01 | 네트워크 | 16,033 |
| 05\_wazuh | wazuh\_alerts | sec-wazuh-01 | 호스트 | 0 |
| 06\_fail2ban | fail2ban | sec-fail2ban-01 | 호스트 | 0 |
| 08\_crowdstrike\_falcon | falcon | sec-web-01 (탐지 대상) | 엔드포인트 | 11 |

※ PHANTOM 실행 로그(공격자 VM, 1,276줄)는 시나리오 정답(ground truth) 기록으로 별도 보관한다.

### 수집 관련 참고

- **Zeek dns·ssl·notice는 0건이다.** 이 구간에 DMZ를 지나는 DNS·TLS 트래픽이 없기 때문이며, 설정 문제가 아니다. Zeek는 체크섬 오프로드 보정(`ignore_checksums`) 적용 후 수집해 conn·http·files·ssh가 정상 기록된다.
- **Wazuh·Fail2ban은 0건이다.** 시나리오 1은 웹 중심 체인이라 SSH 브루트포스·호스트 침입 탐지 규칙을 유발하지 않는다. 해당 알림은 풀체인·시나리오 3에서 발생한다.
- **CrowdStrike Falcon(11건)은 클라우드 등록 시각이 아니라 실제 탐지(event) 시각 기준으로 시나리오 구간만 포함했다.** 등록이 종료 시각 뒤로 지연된 탐지도 탐지 시각이 구간 안이면 포함된다.
- **waf linux\_audit가 6건으로 적은 것**은 audit 이벤트 대부분이 종료 시각 뒤에 몰렸기 때문이다.

## 5. 대표 탐지 규칙 (이번 실행 실측)

이번 수집 구간에서 각 센서가 실제로 발생시킨 상위 탐지 규칙이다. 괄호 안은 발생 건수다.

**ModSecurity CRS (sec-waf-01, waf_modsec)**

| 규칙 ID | 설명 | 건수 |
| --- | --- | ---: |
| 920350 | Host header is a numeric IP address | 7,958 |
| 980130 | Inbound/Outbound anomaly score 집계 | 7,860 |
| 949110 | Inbound Anomaly Score Exceeded (Total Score: 10) | 7,860 |
| 913100 | Found User-Agent associated with security scanner | 4,630 |
| 959100 | Outbound Anomaly Score Exceeded (Total Score: 5) | 3,234 |
| 951230 | MySQL SQL Information Leakage | 3,233 |
| 942100 | SQL Injection Attack Detected via libinjection | 3,073 |
| 942160 | Detects blind sqli tests using sleep()/benchmark() | 1,123 |
| 942190 | Detects MSSQL code execution / info gathering | 507 |
| 932115 | Remote Command Execution: Windows Command Injection | 432 |
| 942140 | SQL Injection Attack: Common DB Names Detected | 346 |

**Suricata (ET 룰셋, dmz·ips·int-server 합산)**

| 시그니처 ID | 시그니처 | 건수 |
| --- | --- | ---: |
| 2016673 | ET WEB_SERVER SQL Errors in HTTP 500 Response | 6,464 |
| 2016935 | ET WEB_SERVER SQL Injection Select Sleep Time Delay | 924 |
| 2017808 | ET WEB_SERVER Possible MySQL SQLi Information Schema | 687 |
| 2011041 | ET WEB_SERVER MYSQL Benchmark Command in URI | 604 |
| 2017337 | ET WEB_SERVER ATTACKER SQLi - SELECT and Schema Columns | 343 |
| 2009358 | ET SCAN Nmap Scripting Engine User-Agent Detected | 22 |
| 2049400 | ET WEB_SERVER /etc/passwd Detected in URI | 15 |
| 2019284 | ET ATTACK_RESPONSE Output of id command from HTTP server | 9 |
| 2010667 | ET WEB_SERVER /bin/bash In URI, Possible Shell Command | 7 |

**CrowdStrike Falcon (sec-web-01 탐지, 총 11건)**

| 전술/기법 | 탐지명 | 건수 |
| --- | --- | ---: |
| TA0006 / T1003 | CredentialAccessLin | 3 |
| TA0006 / T1552 | CredentialTheftLin | 2 |
| TA0003 / T1505.003 | LinWebshell | 2 |
| TA0011 / T1219 | BashReverseShell | 2 |
| TA0003 / T1133 | PersistenceLin | 1 |
| CSTA0001 / CST0001 | PostExploitLin | 1 |

※ Wazuh·Fail2ban은 이번 구간에서 신규 경보가 없어 규칙 샘플이 없다(2절·4절 참고).

## 6. MITRE ATT&CK 매핑

| 전술 | 기법 ID | 기법 | 근거 단계 |
| --- | --- | --- | --- |
| Reconnaissance | T1046 | Network Service Discovery | 1 |
| Reconnaissance | T1595.002 | Active Scanning: Vulnerability Scanning | 1 |
| Initial Access | T1190 | Exploit Public-Facing Application | 2 |
| Execution | T1059.004 | Unix Shell | 3 |
| Credential Access | T1110.001 | Password Guessing | 2 |
| Lateral Movement | T1021.004 | Remote Services: SSH | 4 |
| Collection | T1560.001 | Archive via Utility | 5 |
| Command and Control | T1071.001 | Application Layer Protocol: Web | 6 |
| Command and Control | T1071.004 | Application Layer Protocol: DNS | 6 |
| Exfiltration | T1041 | Exfiltration Over C2 Channel | 7 |
| Exfiltration | T1048.001 | Exfiltration Over Alternative Protocol | 7 |
| Defense Evasion | T1070 | Indicator Removal | 8 |
