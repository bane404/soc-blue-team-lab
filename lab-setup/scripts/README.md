# auditread

A small wrapper around `ausearch` that reformats raw auditd output into a readable, structured event view — filterable by key, user, success/failure, and count.

## Why

Raw `ausearch` output is dense and hard to scan quickly during an investigation — SYSCALL, PATH, CWD, and PROCTITLE records are printed as separate lines that you have to mentally stitch back together per event. `auditread` groups each event into one readable block (when / what / who / where / how / command) so I can scan audit activity faster while working in the lab.

## Usage

```bash
auditread \[-k key] \[-n N] \[-x] \[-s yes|no] \[-u user]
```

|Flag|Meaning|
|-|-|
|`-k key`|filter by audit key (e.g. `shell\_config\_mod`)|
|`-n N`|show N events (newest first by default)|
|`-x`|show oldest first instead of newest|
|`-s yes\|no`|filter by success/failure|
|`-u user`|filter by auid username|

Example:

```bash
auditread -k shell\_config\_mod -n 5
```

## How this was built

I drafted the parsing logic with AI assistance (Claude) rather than writing the `ausearch` output parsing from scratch — the field-extraction regex and event-grouping logic were AI-generated based on a description of what I wanted the tool to do. I tested it against real audit output from the lab, adjusted the filters and output format to match what was actually useful during investigations, and use it regularly as part of the lab workflow.

