# Skills Checklist

I started this lab to actually build things hands-on rather than just read about SOC work — the checklist below is an honest self-assessment, not a list of boxes checked to look complete. Some things I can do independently, some I can do by looking up the exact syntax, and some I haven't really touched yet. I've tried to be precise about which is which rather than rounding everything up to "yes."

Legend: ✅ Can do independently · 🟡 Can do with lookup / partial understanding · ⬜ Not yet

## Networking & Traffic Analysis

- ✅ Read and filter `.pcap` files in Wireshark and tcpdump
- ✅ Identify TCP handshakes, DNS queries, HTTP flows, and ARP in packet captures
- ✅ Identify port scans, SSH brute-force, and beaconing in network logs
- 🟡 Understand Snort/Suricata rule structure at a basic level
- ⬜ Use Zeek to generate and query `conn.log`, `dns.log`, `http.log`
- ⬜ Use `zeek-cut` to extract specific fields from Zeek logs

## Linux Security & Logging

- ✅ Navigate and interpret `/var/log/` (auth.log, syslog, kern.log) — mainly via `journalctl`
- ✅ Harden SSH: key-based auth, disable root login, non-default port
- 🟡 Configure and query auditd for syscall monitoring and file-watch rules — I can do this, but need to look up exact `auditctl`/`ausearch` syntax rather than knowing it cold
- 🟡 Read the systemd journal with `journalctl` and filter by unit/time
- 🟡 Understand Linux file permissions and setuid/setgid in a security context
- 🟡 Explain the order of volatile evidence collection before a reboot

## Windows Logging & Sysmon

- 🟡 Know Event IDs without looking them up — solid on 4624 and 4625, the rest I'd need to check
- 🟡 Identify suspicious process creation chains in Event ID 4688
- 🟡 Read Windows Security/System/Application logs in Event Viewer
- ⬜ Explain what Sysmon adds over native Windows logging

## SIEM — Wazuh

- ✅ Install and configure Wazuh Manager, Indexer, and Dashboard from scratch
- ✅ Deploy Wazuh agents on Linux and Windows — followed official documentation
- ✅ Map Wazuh alerts to MITRE ATT&CK techniques
- 🟡 Read and write Wazuh rule XML (`local_rules.xml`) — two custom rules written and validated (100010, 100020), still building fluency
- 🟡 Configure File Integrity Monitoring (FIM) for critical directories — configured it correctly, but it took a real debugging session to figure out why alerts weren't firing (turned out to be an analysisd overload issue, not the FIM config itself). Worth noting since it's a real diagnostic I worked through, not something I understood on the first try. Full writeup in `investigations/fim-investigation-and-fix.md`
- 🟡 Enable and interpret Vulnerability Detector results — set it up, found and patched a real critical CVE
- ⬜ Build custom dashboards in OpenSearch Dashboards
- ⬜ Convert Sigma rules to Wazuh XML format using sigma-cli — turns out this isn't actually possible, sigma-cli has no Wazuh backend. Translated a rule manually instead; writeup in `sigma-conversions/`

## Threat Intelligence & MITRE ATT&CK

- ✅ Explain the difference between an IOC, a TTP, and a campaign
- 🟡 Navigate ATT&CK Navigator, create and annotate a layer
- 🟡 Map any given event to the correct ATT&CK tactic and technique — I can do this by looking it up, not from memory yet
- 🟡 Use VirusTotal, AbuseIPDB, and Shodan for IOC enrichment — done through TryHackMe rooms, haven't used them independently on real findings yet
- 🟡 Explain the intelligence cycle (Collection → Processing → Analysis → Dissemination)

## DFIR

- 🟡 Collect volatile evidence in the correct order
- 🟡 Write a professional Incident Report with timestamped timeline and MITRE table — three written so far (see `investigations/`), getting more consistent each time
- ⬜ Describe all 6 phases of the NIST IR process — need to review, don't have this memorized
- ⬜ Perform basic disk forensics with Autopsy — haven't done this room yet
- ⬜ Perform basic memory analysis with Redline

## SOC Analyst Process

- ✅ Explain what Tier 1 handles versus what gets escalated to Tier 2
- 🟡 Triage alerts as True Positive / False Positive / Benign with documented reasoning
- 🟡 Distinguish between alert-driven and hypothesis-driven detection
- 🟡 Formulate a threat hunting hypothesis and execute it against SIEM logs
- 🟡 Produce written investigation summaries from scratch — getting there, still developing my own structure/voice for these

## Where I'd focus next

Zeek is the clearest gap — I have Suricata running but never actually built the Zeek habit alongside it. DFIR is the other weak area: I know the theory loosely but haven't done a real disk or memory forensics exercise yet (Autopsy, Redline). Both are reasonable next steps once the current lab work settles.
