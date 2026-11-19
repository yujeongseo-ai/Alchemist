# PHANTOM 데이터셋 — 시나리오 3\
: 인프라 어플라이언스(Fail2ban·Wazuh·WireGuard) 텔레메트리

시나리오 3은 Fail2ban, Wazuh HIDS, WireGuard VPN 3개 독립 보안 어플라이언스를 SSH 스프레이와 핸드셰이크로 자극해 방어 시스템 텔레메트리를 수집한 데이터셋이다.

## 1. 개요

시나리오 3은 2026-11-19 06:43:27Z(KST 15:43)에 시작해 약 3분 55초간 실행된 5단계 시뮬레이션이며, 독립된 보안 어플라이언스 3종(Fail2ban, Wazuh HIDS, WireGuard VPN)의 텔레메트리를 수집했다. 실행 로그는 301줄이고 결과는 13 OK / 5 WARN / 0 FAIL이다. 로그는 SIEM 사본이 아니라 각 보안 장비에서 직접 읽었고, 값은 재작성하지 않았다. 수집 구간은 이벤트 타임스탬프 기준으로 시나리오 시작~종료(06:43:27Z–06:47:22Z)만 남겼다.

| 항목 | 내용 |
| --- | --- |
| 데이터셋 ID | ocsf\_scenario3\_infra\_20261119T064327Z |
| 수행 기관 | CSS LAB (KENTECH) |
| 수집일 | 2026-11-19 |
| 실행 구간 | 06:43:27Z – 06:47:22Z (마지막 기록 06:47:21Z) |
| 시나리오 유형 | SSH 인증 실패 반복, 호스트 이벤트 체인, VPN 핸드셰이크 |
| 단계 수 | 5단계 (STEP 3은 3c–3g 하위 포함) |
| 실행 결과 | 13 OK / 5 WARN / 0 FAIL |
| 수집 소스 | 20개 (7개 센서 패밀리 + CrowdStrike Falcon) |
| 총 용량 | 약 204 KB |

## 2. 수집 환경

어플라이언스 3대와 공격자 관리 호스트가 모두 10.10.50.0/24 대역에 있다. 웹·DB 서비스 없이 방어 장비 자체의 로그만 다룬다.

| 호스트 | IP | 역할 |
| --- | --- | --- |
| 공격자 VM (atk-kali-01) | 10.10.50.80 (관리) | 공격자 관리 호스트 |
| sec-fail2ban-01 | 10.10.50.43 | Fail2ban (sshd jail, sshd-aggressive jail) |
| sec-wazuh-01 | 10.10.50.42 | Wazuh HIDS (탐지·경보 전용, 차단 없음) |
| sec-wireguard-01 | 10.10.50.44 | WireGuard VPN (:22, udp/51820) |
| sec-fw-01 | - | nftables 방화벽 |
| SIEM | 10.10.50.90 | 로그 수집 (:9200) — 이 데이터셋은 SIEM을 거치지 않고 장비 직접 수집 |

## 3. 시나리오 구성

시나리오는 서비스 확인부터 종료 요약까지 5단계로 진행된다. 시각은 UTC 기준 각 단계 시작 시각이다.

| 단계 | 시각 | 이름 | 내용 |
| --- | --- | --- | --- |
| 1 | 06:43:30 | 서비스 확인 | 어플라이언스 3대의 SSH(:22)와 WireGuard udp/51820 포트 확인 |
| 2 | 06:43:40 | Fail2ban SSH 인증 실패 반복 | sshd·sshd-aggressive 두 jail을 대상으로 SSH 자격증명 스프레이 |
| 3 | 06:44:43 | Wazuh HIDS 이벤트 체인 | 인증 스프레이(2개 규칙 유형) + 정상 로그인 + syscheck 등 하위 체인 |
| 4 | 06:45:35 | WireGuard 핸드셰이크 | 클라이언트 인터페이스를 올려 실제 피어 핸드셰이크 3회 사이클 |
| 5 | 06:47:21 | 종료 요약 | 3종 어플라이언스 실행 완료 기록 |

STEP 3의 하위 단계(실행 로그 기준):

| 하위 | 시각 | 내용 |
| --- | --- | --- |
| 3(초반) | 06:44:43 | 존재하지 않는 계정/실제 계정 대상 인증 실패 스프레이(2개 규칙 유형) |
| 3c | 06:45:16 | 공개키 인증 로그인 성공(실패 후 성공 상관) |
| 3d | 06:45:17 | 로그인 후 계정·프로세스·파일 조회(의심 recon) |
| 3e | 06:45:17 | 루트킷 회피 점검(rootkit-evasion probe) |
| 3f | 06:45:18 | /etc 감시 경로 파일 추가·삭제(syscheck FIM) |
| 3g | 06:45:27 | 로그 변조 시도(log tampering probe) |

WireGuard 핸드셰이크는 3회 사이클(cycle 1~3)로 실행되었고 각 사이클에서 핸드셰이크가 성립했다(핑은 100% 손실로 설계됨).

## 4. 수집 장비 및 로그 소스

로그는 방어 어플라이언스와 네트워크 센서에서 수집했다. 아래 줄 수는 이번 실행 구간에 실제로 수집된 양이다.

| 패밀리 | 소스 | 호스트 | 계층 | 줄 수 |
| --- | --- | --- | --- | ---: |
| 05\_wazuh | wazuh\_alerts | sec-wazuh-01 | 호스트 | 62 |
| 06\_fail2ban | fail2ban | sec-fail2ban-01 | 호스트 | 12 |
| 04\_nftables\_firewall | kernel | sec-fw-01 | 네트워크 | 262 |
| 03\_waf\_modsecurity | linux\_audit | sec-waf-01 | 호스트 | 21 |
| 02\_zeek | zeek\_conn | sec-ids-dmz-01 | 네트워크 | 22 |
| 01\_suricata | suricata\_eve | sec-ips-01 | 네트워크 | 20 |
| 01\_suricata | suricata\_eve | sec-ids-dmz-01 | 네트워크 | 17 |
| 01\_suricata | suricata\_eve | sec-ids-int-server-01 | 네트워크 | 1 |
| 03\_waf\_modsecurity | waf\_access / waf\_modsec | sec-waf-01 | 애플리케이션 | 0 |
| 08\_crowdstrike\_falcon | falcon | - | 엔드포인트 | 0 |
| 02\_zeek | zeek\_dns / http / ssh / files / ssl 등 | sec-ids-dmz-01 | 네트워크 | 0 |

※ PHANTOM 실행 로그(공격자 VM, 301줄)는 시나리오 정답(ground truth) 기록으로 별도 보관한다. WireGuard 핸드셰이크 상세(사이클별 전송량 등)는 이 실행 로그에 남는다.

### 수집 관련 참고

- **시나리오 3은 Wazuh·Fail2ban이 핵심 소스다.** 세 방어 어플라이언스를 직접 자극하는 체인이라, 시나리오 1·2와 달리 Wazuh(62건)·Fail2ban(12건)에서 가장 많은 탐지가 나온다. 웹·EDR 소스(WAF web, Falcon)는 0건이다.
- **WireGuard는 OCSF 매핑 대상이 아니라 데이터셋에서 제외했다.** 매핑 클래스가 없는 소스이며(ocsf\_mapping.csv 기준), VPN 활동은 핸드셰이크 자체로만 실행 로그에 남는다. STEP 4가 생성한 핸드셰이크 텔레메트리는 OCSF 소스 목록에 포함되지 않는다.
- **Falcon(0건)**은 이 시나리오가 EDR 호스트(sec-web-01, 클라이언트)를 건드리지 않기 때문이다.

### 대표 탐지 규칙 (이번 실행 실측)

**Wazuh (sec-wazuh-01, 총 62건)**

| 규칙 ID | 레벨 | 설명 | 건수 |
| --- | ---: | --- | ---: |
| 5501 | 3 | PAM: Login session opened | 13 |
| 5502 | 3 | PAM: Login session closed | 12 |
| 5710 | 5 | sshd: Attempt to login using a non-existent user | 11 |
| 5503 | 5 | PAM: User login failed | 7 |
| 40112 | 12 | Multiple authentication failures followed by success | 6 |
| 5402 | 3 | Successful sudo to ROOT executed | 6 |
| 5760 | 5 | sshd: authentication failed | 4 |
| 5712 | 10 | sshd: brute force trying to get access | 1 |
| 5551 | 10 | PAM: Multiple failed logins in a small period | 1 |
| 92604 | 6 | Processes running for all users were queried | 1 |

**Fail2ban (sec-fail2ban-01, 총 12줄)**: `[sshd]` jail Ban 2회, Unban 1회 + Found/Notice 기록.

## 5. MITRE ATT&CK 매핑

| 전술 | 기법 ID | 기법 | 근거 단계 |
| --- | --- | --- | --- |
| Credential Access | T1110.001 | Brute Force: Password Guessing | 2, 3 |
| Credential Access | T1110.003 | Brute Force: Password Spraying | 2, 3 |
| Initial Access | T1078 | Valid Accounts | 3c |
| Discovery | T1087 | Account Discovery | 3d |
| Discovery | T1057 | Process Discovery | 3d |
| Defense Evasion | T1562.001 | Impair Defenses: Disable or Modify Tools | 3e |
| Defense Evasion | T1070.002 | Indicator Removal: Clear Linux or Mac System Logs | 3g |
| Persistence | T1098.004 | Account Manipulation: SSH Authorized Keys | 3f |
| Command and Control | T1572 | Protocol Tunneling (WireGuard) | 4 |
