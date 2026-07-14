# **Manual Sigma-to-Wazuh Rule Conversion**

**Environment:** Debian 12 (192.168.1.13), Wazuh 4.14.6 (all-in-one), sigma-cli


**Analyst:** Abderrahim ENASSIBI


**Date:** 2026-07-13

\---

## **Summary**

The plan was to pull a Sigma rule from the official SigmaHQ repo and convert it to Wazuh XML using `sigma-cli`. That tool doesn't actually support Wazuh as a conversion target — confirmed directly rather than assumed — so the rule was translated by hand instead. Ended up being a better exercise for actually understanding rule logic than a straight automated conversion would've been.

## **Confirming there's no Wazuh backend**

```
$ sigma plugin list
```

Full plugin list includes backends for QRadar, Cortex XDR, Carbon Black, SentinelOne, Splunk, Elasticsearch, OpenSearch, CrowdStrike, Kusto, Datadog, Panther, Logpoint, SQLite, and about a dozen more. No `wazuh` entry anywhere in the list.

Wazuh's indexer is built on OpenSearch, and sigma-cli does support an OpenSearch backend — but that produces OpenSearch alerting-rule syntax, not Wazuh's own `local\_rules.xml` rule format. The two aren't interchangeable; Wazuh detection rules and OpenSearch dashboards/alerts are different systems under the same platform. So even the closest available backend doesn't actually solve the problem.

## **The rule**

Source: `sigma/rules/linux/auditd/path/lnx\_auditd\_unix\_shell\_configuration\_modification.yml` from the SigmaHQ repository — detects modification of shell configuration files, a common persistence technique where an attacker who's gained access appends commands that execute every time a new shell is opened.

**Original Sigma rule:**

```yaml
title: Unix Shell Configuration Modification
id: a94cdd87-6c54-4678-a6cc-2814ffe5a13d
status: test
description: Detect unix shell configuration modification. Adversaries may establish persistence through executing malicious commands triggered when a new shell is opened.
author: Peter Matkovski, IAI
date: 2023-03-06
modified: 2023-03-15
tags:
    - attack.privilege-escalation
    - attack.persistence
    - attack.t1546.004
logsource:
    product: linux
    service: auditd
detection:
    selection:
        type: 'PATH'
        name:
            - '/etc/shells'
            - '/etc/profile'
            - '/etc/profile.d/\*'
            - '/etc/bash.bashrc'
            - '/etc/bashrc'
            - '/etc/zsh/zprofile'
            - '/etc/zsh/zshrc'
            - '/etc/zsh/zlogin'
            - '/etc/zsh/zlogout'
            - '/etc/csh.cshrc'
            - '/etc/csh.login'
            - '/root/.bashrc'
            - '/root/.bash\_profile'
            - '/root/.profile'
            - '/root/.zshrc'
            - '/root/.zprofile'
            - '/home/\*/.bashrc'
            - '/home/\*/.zshrc'
            - '/home/\*/.bash\_profile'
            - '/home/\*/.zprofile'
            - '/home/\*/.profile'
            - '/home/\*/.bash\_login'
            - '/home/\*/.bash\_logout'
            - '/home/\*/.zlogin'
            - '/home/\*/.zlogout'
    condition: selection
falsepositives:
    - Admin or User activity are expected to generate some false positives
level: medium
```

Note the scope difference: the Sigma rule watches roughly two dozen shell config paths across bash, zsh, and csh, system-wide and per-user. For the lab, this was narrowed to a single path relevant to the actual environment (`/home/abdo/.bashrc`) rather than replicating the full path list — a deliberate scoping decision for a single-user lab, not a limitation of the translation itself.

**Manual translation to Wazuh XML** (`local\_rules.xml`, rule 100020):

```xml
<group name="local,audit,">
  <rule id="100020" level="7">
    <if\_sid>80700</if\_sid>
    <field name="audit.key">shell\_config\_mod</field>
    <description>Shell configuration file modified: $(audit.file.name)</description>
    <mitre>
      <id>T1546.004</id>
    </mitre>
    <group>persistence,local\_rules,</group>
  </rule>
</group>
```

The Sigma rule matches on an explicit list of file paths. Wazuh's approach here is different — rather than pattern-matching a path at rule-evaluation time, the audit watch itself is scoped to the target file with a specific key:

```bash
auditctl -w /home/abdo/.bashrc -p wa -k shell\_config\_mod
```

The Sigma rule does its filtering right there in the detection block — it lists every path to watch. Wazuh splits that in two: the path-watching happens earlier, in the auditd config itself (auditctl -w /home/abdo/.bashrc -p wa -k shell\_config\_mod), and the Wazuh rule just checks for that key showing up in the logs. So instead of one rule doing everything, the "what to watch" part and the "alert on it" part live in two different places. That's really what the manual conversion came down to — not translating line by line, but figuring out which platform expects which part of the logic.

## **Validation**

Triggered by appending to the watched file and confirming the alert fired with the correct rule ID, MITRE mapping, and file path in the alert body. Rule fires as expected.

## **Takeaway**

Automated conversion tools are only as good as their backend coverage — worth checking `sigma plugin list` (or equivalent) before assuming a conversion path exists, rather than debugging a command that was never going to work. Doing the translation by hand also forced a clearer understanding of how Wazuh's rule-matching model (base rule + field/key matching) differs structurally from Sigma's declarative field-matching model, which a working `sigma convert` command would have hidden.

