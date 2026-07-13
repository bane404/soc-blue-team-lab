# **Incident Report — SSH Brute Force to Successful Compromise**

**Environment:** Debian 12 (192.168.1.13), Wazuh 4.14.6 (all-in-one), Suricata 7.0.10
**Analyst:** Abderrahim ENASSIBI
**Date:** 2026-07-12
**Classification:** Authorized internal lab exercise

\---

## **Summary**

I ran a self-directed attack simulation against my own lab server to test whether the SIEM/IDS stack would actually catch a full attack chain: recon, brute force, login, post-access enumeration. It caught three of the four stages cleanly. One custom detection rule failed to fire, which turned out to be a useful finding on its own. The test credential was reset immediately after.

## **Scope and Authorizatio**n

Self-owned infrastructure, no third parties or production systems involved. The account password was deliberately set to a weak value (`00000000`) for a bounded window to validate brute-force detection, then reset to a strong password right after.

## **Timeline** 

|Time|Event|
|-|-|
|17:56|Initial nmap scan (`-sV -A`) against 192.168.1.13|
|18:09|Second nmap scan against the same target|
|19:19:25|Hydra SSH brute-force started (rockyou.txt, 16 threads)|
|19:24:14|Hydra recovers valid credential (`abdo:00000000`)|
|19:28:13|Successful SSH login with the cracked credential|
|\~19:30|Post-login enumeration (`whoami`, `id`, `uname -a`, `ps aux`, `ss -tulnp`)|
|\~19:31|Password reset, exercise closed out|

## **Technical Analysis**

### Reconnaissance

Two `nmap -sV -A` scans from an external Kali host. Suricata flagged the Nmap Scripting Engine user-agent string during service probing, and separately flagged probe traffic on port 4444 as trojan-category due to signature overlap with known Metasploit handler traffic. Both were ingested and correctly classified by Wazuh.

### Brute Force

Hydra against SSH with the rockyou.txt wordlist, 16 parallel threads. The target password (line 907 of the wordlist) was recovered in under five minutes. Wazuh's built-in correlation rule (40112) caught the failed-then-successful pattern and generated a single alert covering both Brute Force and Valid Accounts.

A custom rule I'd built earlier (100010) — meant to escalate on repeated matches of Wazuh's base SSH brute-force rule (5712) within a 10-minute window — never fired. This wasn't a config mistake: rule 40112 already caught the pattern independently, so nothing was missed operationally. But it points to a real gap in how 100010 was tuned. Hydra's fast, parallel attack likely didn't generate two distinct 5712 events inside the matching window before the correct password was found — the rule's timing assumptions were built around a slower, sequential brute-force pattern, not a fast automated one. Worth revisiting as a tuning exercise.

### Initial Access

Login was logged and attributed correctly by the same rule 40112.

### Discovery

Post-login enumeration commands didn't trigger anything — expected, since there's no file integrity or audit watch on those paths and the session was already authenticated. Noted as a coverage boundary, not a failure.

## Evidence

* Suricata sig 2024364 — "ET SCAN Possible Nmap User-Agent Observed" (Wazuh rule 86601)
* Suricata sig 3400020 — "POSSBL SCAN SHELL M-SPLOIT TCP", port 4444 (incidental to nmap probing)
* Wazuh rule 40112 — multiple auth failures followed by success (level 12): `Accepted password for abdo from 192.168.1.1 port 47506 ssh2`
* Hydra output confirming the recovered credential
* Manual enumeration output captured via screenshot

## **MITRE ATT\&CK Mapping**

|Tactic|Technique|Evidence|
|-|-|-|
|Reconnaissance|T1046 — Network Service Discovery|Suricata 2024364, 3400020 → Wazuh 86601|
|Credential Access|T1110.001 — Brute Force: Password Guessing|Wazuh 40112|
|Initial Access|T1078 — Valid Accounts|Wazuh 40112|
|Discovery|T1082 — System Information Discovery|Manual observation, not alert-generating under current config|

## Recommendations

1. Move SSH to key-based auth and disable password login — attempted separately, reverted for now (see hardening notes).
2. Re-tune rule 100010's frequency/timeframe thresholds for fast parallel-threaded attacks, or retire it in favor of relying on 40112.
3. Consider audit watches or command-monitoring on sensitive post-login commands (`ps`, `ss`, `netstat`) if that visibility is worth the added noise.
4. Rate-limit or fail2ban SSH to shrink the practical window for automated credential attacks regardless of password strength.

## Severity

**High as simulated** — a real-world equivalent (valid SSH access, especially with sudo membership) would be a critical initial-access foothold. Actual risk here was nil: the weak credential was intentional, time-boxed, and reset immediately after.

