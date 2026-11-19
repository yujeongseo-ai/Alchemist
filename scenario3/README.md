# PHANTOM scenario3_infra — OCSF mapping source logs

Scenario window (UTC): `2026-11-19T06:43:27+00:00` to `2026-11-19T06:47:22+00:00`. Only events inside this window are kept.

device-direct append window since byte mark (no SIEM copy, no value rewriting); zeek uses ts>=scenario start; wireguard excluded (not OCSF source); all VM clocks synced to UTC before run; trimmed to scenario window 2026-11-19T06:43:27+00:00 <= event timestamp <= 2026-11-19T06:47:22+00:00 (timezone offsets honored; lines without a timestamp follow the preceding line)

| family | source | lines | bytes |
|---|---|---:|---:|
| 01_suricata | sec-ids-dmz-01/suricata_eve | 17 | 10141 |
| 01_suricata | sec-ips-01/suricata_eve | 20 | 12306 |
| 01_suricata | sec-ids-int-server-01/suricata_eve | 1 | 536 |
| 01_suricata | sec-ids-int-user-01/suricata_eve | 0 | 0 |
| 01_suricata | sec-ids-decoy-01/suricata_eve | 0 | 0 |
| 02_zeek | sec-ids-dmz-01/zeek_conn | 22 | 9379 |
| 02_zeek | sec-ids-dmz-01/zeek_dns | 0 | 0 |
| 02_zeek | sec-ids-dmz-01/zeek_http | 0 | 0 |
| 02_zeek | sec-ids-dmz-01/zeek_files | 0 | 0 |
| 02_zeek | sec-ids-dmz-01/zeek_ssh | 0 | 0 |
| 02_zeek | sec-ids-dmz-01/zeek_ssl | 0 | 0 |
| 02_zeek | sec-ids-dmz-01/zeek_notice | 0 | 0 |
| 02_zeek | sec-ids-dmz-01/zeek_weird | 0 | 0 |
| 03_waf_modsecurity | sec-waf-01/waf_access | 0 | 0 |
| 03_waf_modsecurity | sec-waf-01/waf_modsec | 0 | 0 |
| 03_waf_modsecurity | sec-waf-01/linux_audit | 21 | 4178 |
| 04_nftables_firewall | sec-fw-01/kernel | 262 | 67237 |
| 05_wazuh | sec-wazuh-01/wazuh_alerts | 62 | 60595 |
| 06_fail2ban | sec-fail2ban-01/fail2ban | 12 | 1274 |
| 08_crowdstrike_falcon | crowdstrike/falcon | 0 | 0 |
