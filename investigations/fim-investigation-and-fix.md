# File Integrity Monitoring — Investigation and Fix

**Environment:** Debian 12 (192.168.1.13), Wazuh 4.14.6 (all-in-one)
**Analyst:** Abderrahim ENASSIBI
**Date:** 2026-07-14

\---

## Summary

FIM alerts weren't firing reliably despite a correct `syscheck` config. Tracked it down to `wazuh-analysisd` silently dropping events because of a JSON decoder field limit, triggered by Suricata's verbose `eve.json` output overwhelming the manager. Raised the decoder limit, confirmed both general FIM (rule 550) and a custom auditd-based detection rule (100020) now fire correctly and reliably.

## Starting point

`syscheck` was configured correctly from early on — realtime watches on `/etc`, `/usr/bin`, `/home`, `scan\_on\_start` enabled — but test file changes weren't producing alerts. No obvious config error, which made it a genuinely confusing one to debug.

## Diagnostic steps

**Confirmed syscheck itself was running and had a real baseline.** Queried the FIM database directly:

```bash
sudo sqlite3 /var/ossec/queue/fim/db/fim.db "SELECT COUNT(\*) FROM file\_entry;"
```

Returned 8,586 entries — syscheck was scanning and tracking files correctly. The problem wasn't the scanning side.

**Turned on debug logging and watched a live scan.** With `syscheck.debug=2` set and the manager restarted, the log showed syscheck actively sending integrity-check messages for its baseline scan — working as expected. But interleaved with those messages was a wall of:

```
wazuh-analysisd: ERROR: Too many fields for JSON decoder.
```

This is `wazuh-analysisd` — the component that turns incoming events into alerts — rejecting messages because they exceeded its default field limit (256 fields per JSON decoder, per Wazuh's internal options). It wasn't specific to syscheck's messages; anything with enough fields was getting dropped.

**Traced the actual source.** This is a known issue tied specifically to Suricata's `eve.json` integration — Suricata logs verbose per-flow metadata (HTTP, TLS, flow stats) that can exceed the decoder's field ceiling. Since `eve.json` ingestion was added into `ossec.conf` earlier in the lab, that's almost certainly what was periodically overloading analysisd and disrupting its ability to keep up with other event processing, FIM alerts included.

## Fix

Raised the decoder field limit in `/var/ossec/etc/local\_internal\_options.conf`:

```
analysisd.decoder\_order\_size=1024
```

Restarted the manager. Multiple sources online report that this alone doesn't always fully eliminate the error under sustained load (the real long-term fix is trimming Suricata's `eve.json` fields at the source), but it was enough to clear the backlog and restore alerting for a normal workload.

## Verification

**General FIM (rule 550):** confirmed via the dashboard — alert history for rule 550 goes back to before this fix, meaning baseline FIM detection was already working intermittently. Two additional test changes to `/etc/hosts` produced clean, complete alerts with full before/after checksums (md5, sha1, sha256) and correct MITRE mapping (T1565.001).

**Custom rule 100020 (shell config modification):** this was the actual unresolved gap — the rule had never fired, ever, despite the auditd watch (`auditctl -w /home/abdo/.bashrc -p wa -k shell\_config\_mod`) being correctly registered. Traced the full chain manually:

1. `sudo ausearch -k shell\_config\_mod` — confirmed auditd logged the syscall against `.bashrc` with the correct key
2. Checked Wazuh's ingestion of `/var/log/audit/audit.log` via the `auditd` decoder
3. Checked `alerts.json` directly for a match

After a real trigger (appending a line to `.bashrc`), the alert fired correctly:

```json
"rule":{"level":7,"description":"Shell configuration file modified: /home/abdo/.bashrc","id":"100020",
"mitre":{"id":\["T1546.004"],"tactic":\["Privilege Escalation","Persistence"]}}
```

Full chain confirmed end to end: audit watch → auditd log entry -> Wazuh's auditd decoder -> custom rule match -> alert with correct MITRE mapping.

## What this actually was

Not a syscheck configuration problem. The root cause was `analysisd` getting overloaded by oversized JSON from the Suricata integration, which appears to have delayed or dropped processing across multiple event sources — not just FIM. The fix (raising the decoder limit) addressed the symptom well enough to restore reliable alerting; the proper long-term fix is reducing the size of what Suricata sends `eve.json` in the first place, which hasn't been done yet and is a legitimate open item.

## Open item

Suricata's `eve.json` output is still verbose enough to occasionally hit the decoder ceiling under bursty traffic (confirmed by a large error count during one test session). Trimming the `eve-log` field set in `/etc/suricata/suricata.yaml` to essentials (alert metadata, minimal flow/HTTP/TLS fields) is the more durable fix and hasn't been implemented yet.

