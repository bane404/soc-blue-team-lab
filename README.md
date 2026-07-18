# SOC Home Lab

A self-built SOC/blue team lab I put together to get real hands-on experience before going into SOC analyst roles. Debian server running Wazuh and Suricata, a Windows agent, and a series of attack simulations and investigations against it.

I'm a cybersecurity engineering student at ENSA Tétouan, aiming for entry-level SOC/blue team roles. This lab (and the writeups in it) is how I taught myself the actual day-to-day of the job instead of just reading about it.

📄 **[Full report (PDF)](docs/SOC-Home-Lab-Report.pdf)** — the complete writeup with timelines, MITRE mappings, and rule XML.

## What's actually in here

Everything below is real — real config, real alerts, real bugs I hit and had to debug. Where something didn't work the way it was supposed to, I wrote that up too instead of only showing the clean parts. I think that's more useful to read than a portfolio that pretends nothing ever broke.


## Architecture

!\[Network diagram](lab-setup/network-diagram.png)

Debian server (`192.168.1.13`) running Wazuh manager, indexer, and dashboard, plus Suricata for network IDS. A Windows machine runs the Wazuh agent. Attack simulations were run from a Kali live-boot USB on the same network.

One thing worth calling out on the diagram itself: traffic from the attacker box gets NAT'd by the router before it reaches the server, so Wazuh/Suricata alerts show the router's IP as the source instead of the actual attacker IP. Took a bit to figure out why the source IP in alerts didn't match what I expected — documented as a lab topology quirk, not a detection failure.

## Writeups

* [**SSH brute force → compromise**](investigations/incident-report-ssh-bruteforce.md) — full attack chain from recon to login, what got caught and what didn't
* [**FIM investigation and fix**](investigations/fim-investigation-and-fix.md) — File Integrity Monitoring wasn't alerting despite correct config; root cause turned out to be an unrelated component silently dropping events under load. Probably the most useful writeup in here if you want to see actual debugging, not just a config walkthrough.
* [**SSH hardening and vulnerability detection**](investigations/ssh-hardening-and-vulnerability-detection.md) — hardening decisions (including one I tried, hit friction with, and reverted), plus a real critical CVE found and patched
* [**Manual Sigma-to-Wazuh conversion**](sigma-conversions/sigma-to-wazuh-conversion.md) — sigma-cli doesn't actually support Wazuh as a conversion target, so I translated a rule by hand and wrote up why the tool didn't work rather than assuming I was doing something wrong

## Structure

```
soc-blue-team-lab/
├── investigations/       incident reports and technical writeups
├── custom-rules/         local\_rules.xml — custom Wazuh detection rules
├── sigma-conversions/    manual Sigma-to-Wazuh rule translation + notes
└── lab-setup/
    ├── network-diagram.png / .drawio
    ├── agent-config/      sanitized ossec.conf
    └── scripts/           auditread — a small audit log formatting tool
```

## Custom detection rules

Two rules currently live in `custom-rules/local\_rules.xml`:

* **100010** — escalates on repeated SSH brute-force detections from the same source (T1110.001). Turned out to have a real tuning gap — didn't fire against a fast, parallel-threaded Hydra attack because Wazuh's built-in correlation rule already caught it through a different path. Documented in the incident report rather than quietly fixed, since it's a legitimate finding about rule design.
* **100020** — detects modification of shell config files as a persistence indicator (T1546.004), manually translated from a SigmaHQ rule since no Sigma-to-Wazuh backend exists.

## What I'd still like to add

* Trimming Suricata's `eve.json` output — it's verbose enough to occasionally overload the analysis engine under bursty traffic, which was part of the FIM issue above. Raised a decoder limit as a workaround; the more correct fix is reducing what Suricata logs in the first place.
* More BTLO/CyberDefenders writeups — a few are linked from my [security-writeups](https://github.com/bane404/security-writeups) repo, separate from this one.


## Contact

[LinkedIn](https://www.linkedin.com/in/abderrahim-enassibi-19a3a5388/)

