# Wazuh SIEM Lab

A self-hosted Wazuh (open-source SIEM/XDR) lab built to learn and demonstrate
practical detection engineering, log ingestion, and incident response —
covering endpoint monitoring, cloud (AWS CloudTrail) log ingestion, File
Integrity Monitoring, and automated Active Response.

This repo documents the environment, what was configured, and the problems hit along the way and how they were
diagnosed.

## Architecture

```
                    ┌─────────────────────────────┐
                    │   AWS EC2 (Docker host)     │
                    │  ┌───────────────────────┐  │
                    │  │  Wazuh Manager          │  │
                    │  │  Wazuh Indexer          │──┼──── CloudTrail logs
                    │  │  Wazuh Dashboard        │  │        (via S3)
                    │  └───────────────────────┘  │
                    └──────────────┬──────────────┘
                                   │  agent traffic (1514/1515)
                         ┌─────────┴─────────┐
                         │                   │
            ┌────────────┴──────────┐ ┌──────┴─────────────────┐
            │  RHEL 8.10 VM          │ │  Windows 11 PC          │
            │  (RHEL-8.10-VM)        │ │  (AdamsLaptop)          │
            │  Wazuh Agent           │ │  Wazuh Agent + Apache   │
            │  - "Attacker" role     │ │  - "Victim" web server  │
            │  - Log collection      │ │  - FIM (Downloads)      │
            │  - Bridged networking  │ │  - Active Response      │
            │                        │ │    target (netsh.exe)   │
            └────────────────────────┘ └─────────────────────────┘
```

- **Manager stack**: Wazuh (manager + indexer + dashboard) via Docker Compose, on an AWS EC2 instance.
- **Cloud visibility**: AWS CloudTrail → S3 → Wazuh `aws-s3` wodle, so IAM/API activity in the AWS account itself is monitored, not just endpoint activity.
- **Endpoints**: two agents report to the manager —
  - **RHEL 8.10 VM** (`RHEL-8.10-VM`) — runs on VMware Workstation with bridged networking, plays the "attacker" role generating test traffic.
  - **Windows 11 PC** (`AdamsLaptop`) — runs Apache as the "victim" web server, used for FIM testing (`Downloads` folder monitoring) and as the Active Response target for the custom detection rule.

## What's implemented

| Capability | Status | Doc |
|---|---|---|
| Manager stack on AWS (Docker Compose) | ✅ Working | [`docs/01-setup-and-operations.md`](docs/01-setup-and-operations.md) |
| CloudTrail → S3 → Wazuh ingestion | ✅ Working | [`docs/01-setup-and-operations.md`](docs/01-setup-and-operations.md) |
| Agent enrollment (RHEL VM + Windows PC) | ✅ Working | [`docs/01-setup-and-operations.md`](docs/01-setup-and-operations.md) |
| File Integrity Monitoring (FIM) | ✅ Working | [`docs/02-detection-capabilities.md`](docs/02-detection-capabilities.md) |
| Custom detection rule + Active Response trigger | ✅ Working (detection + trigger both fire correctly) | [`docs/02-detection-capabilities.md`](docs/02-detection-capabilities.md) |
| Active Response actual block (Windows `netsh.exe`) | ⚠️ Blocked by upstream bug (confirmed) | [`docs/04-known-issues.md`](docs/04-known-issues.md) |
| S3 bucket versioning | ✅ Enabled | [`docs/03-environment-audit.md`](docs/03-environment-audit.md) |
| S3 Object Lock | 🔜 Planned (prerequisite done) | [`docs/03-environment-audit.md`](docs/03-environment-audit.md) |

The Active Response item is intentionally left showing the "⚠️" status
rather than papered over — see that doc for the full root-cause writeup.

## Repo structure

```
wazuh-siem-lab/
├── README.md                       ← you are here
└── docs/
    ├── 01-setup-and-operations.md  AWS instance sizing, Docker Compose deployment, CloudTrail integration,
    │                               agent enrollment, restart procedure, command reference
    ├── 02-detection-capabilities.md  FIM setup + custom detection rule/Active Response case study,
    │                                 with full root-cause debugging logs
    ├── 03-environment-audit.md     Living audit log: environment snapshot, issues found and resolved,
    │                               IAM/S3 hardening decisions
    └── 04-known-issues.md          Unresolved upstream bugs — root-caused, reported, not yet fixed
```

## Key things I learned building this

A few of the more useful, non-obvious lessons (full detail in the linked docs):

- **Wazuh's Docker Compose deployment uses named volumes, not bind mounts** — editing the repo's local `config/` folder after first startup has no effect on the running stack; you have to edit inside the container (or the manager's config) directly.
- **Instance sizing matters more than expected** — the indexer's JVM heap alone needs enough headroom that anything under ~4GB RAM risks OOM kills severe enough to freeze SSH, not just the container.
- **`/root/.aws/credentials` isn't in a named volume** — it survives a plain restart but is lost on `--force-recreate` or a full stack rebuild, unlike `/var/ossec/etc/`. Learned this the hard way when it caused a real CloudTrail ingestion outage — see the audit doc's Issue 1.
- **A rule can fire and an Active Response command can execute successfully, and the actual remediation can still fail** — worth checking the command's own log output (`active-responses.log`), not just whether Wazuh says it ran.
- **Wazuh agent auto-enrollment is on by default**, and falls back to the machine's hostname if the agent's connection state is ever disrupted — caused two separate "phantom duplicate agent" bugs on Windows and Linux, each with a different root cause (XML nesting for the enrollment block vs. `agent-auth` defaulting to hostname). Full writeup in the audit doc.
- **An IAM identity can read an S3 object without being allowed to read the bucket's own configuration** (policy, versioning, public access block) — these are separate permission sets, a good real-world example of least privilege in practice.

## Next steps

- [ ] S3 Object Lock on the CloudTrail bucket (versioning prerequisite already enabled)
- [ ] Write a custom PowerShell Active Response script to replace the bundled `netsh.exe` binary (parse alert JSON directly, issue `netsh advfirewall` command myself)
- [ ] Test the Linux equivalent (`firewall-drop.sh`) to confirm whether the JSON-parsing bug is Windows-binary-specific
- [ ] Map existing custom rules to MITRE ATT&CK technique IDs
- [ ] Add Suricata/Zeek for network-layer detection alongside host-based FIM/log detections
- [ ] Separate read-only vs. write/hardening IAM users (considered, currently combined for this lab's risk profile — see audit doc)

## Disclaimer

All IPs, bucket names, account IDs, and keys in this repo are placeholders.
This is a personal lab environment, not a production deployment.
