# Wazuh SIEM Lab

A self-hosted Wazuh (open-source SIEM/XDR) lab built to learn and demonstrate
practical detection engineering, log ingestion, and incident response —
covering endpoint monitoring, cloud (AWS CloudTrail) log ingestion, File
Integrity Monitoring, and automated Active Response.

This repo documents the environment, what was configured, and — just as
importantly — the real problems hit along the way and how they were
diagnosed. Most tutorials show the happy path; this is closer to what
actually happens when you build one of these yourself.

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
                                   │
                    ┌──────────────┴──────────────┐
                    │   RHEL 8.10 VM (endpoint)    │
                    │   Wazuh Agent                │
                    │   - FIM (syscheck)            │
                    │   - Log collection            │
                    │   - Active Response target    │
                    └───────────────────────────────┘
```

- **Manager stack**: Wazuh (manager + indexer + dashboard) via Docker Compose, on an AWS EC2 instance.
- **Cloud visibility**: AWS CloudTrail → S3 → Wazuh `aws-s3` wodle, so IAM/API activity in the AWS account itself is monitored, not just endpoint activity.
- **Endpoint**: RHEL 8.10 VM running the Wazuh agent, used for FIM testing, log-based detections, and Active Response.

## What's implemented

| Capability | Status | Doc |
|---|---|---|
| Manager stack on AWS (Docker Compose) | ✅ Working | [`docs/01-aws-docker-setup.md`](docs/01-aws-docker-setup.md) |
| CloudTrail → S3 → Wazuh ingestion | ✅ Working | [`docs/01-aws-docker-setup.md`](docs/01-aws-docker-setup.md) |
| Agent enrollment (Linux + Windows) | ✅ Working | [`docs/02-add-agent.md`](docs/02-add-agent.md) |
| File Integrity Monitoring (FIM) | ✅ Working | [`docs/03-fim-setup.md`](docs/03-fim-setup.md) |
| Custom detection rule + Active Response trigger | ✅ Working (detection + trigger both fire correctly) | [`docs/04-active-response-blocking.md`](docs/04-active-response-blocking.md) |
| Active Response actual block (Windows `netsh.exe`) | ⚠️ Blocked by upstream bug | [`docs/04-active-response-blocking.md`](docs/04-active-response-blocking.md) |
| Local demo environment (Docker Desktop + VM) | ✅ Working | [`docs/06-demo-restart-guide.md`](docs/06-demo-restart-guide.md) |

The Active Response item is intentionally left showing the "⚠️" status
rather than papered over — see that doc for the full root-cause writeup.
Being able to say *"I traced this to a bug in Wazuh's compiled Windows
binary, confirmed it via the binary's own log output, and here are three
ways I'd resolve it"* is a stronger portfolio signal than a repo where
everything suspiciously works on the first try.

## Repo structure

```
wazuh-siem-lab/
├── README.md                          ← you are here
└── docs/
    ├── 01-aws-docker-setup.md         AWS instance sizing, Docker Compose deployment, CloudTrail integration
    ├── 02-add-agent.md                Agent enrollment (Linux auto-enroll vs. Windows manual key)
    ├── 03-fim-setup.md                File Integrity Monitoring config and testing
    ├── 04-active-response-blocking.md Custom detection rule + Active Response, with full root-cause debugging log
    ├── 05-command-reference.md        Running command reference across AWS/Docker/Wazuh
    └── 06-demo-restart-guide.md       Local demo environment (Docker Desktop + VM) setup and restart procedure
```

## Key things I learned building this

A few of the more useful, non-obvious lessons (full detail in the linked docs):

- **Wazuh's Docker Compose deployment uses named volumes, not bind mounts** — editing the repo's local `config/` folder after first startup has no effect on the running stack; you have to edit inside the container (or the manager's config) directly.
- **Instance sizing matters more than expected** — the indexer's JVM heap alone needs enough headroom that anything under ~4GB RAM risks OOM kills severe enough to freeze SSH, not just the container.
- **`/root/.aws/credentials` isn't in a named volume** — it survives a plain restart but is lost on `--force-recreate` or a full stack rebuild, unlike `/var/ossec/etc/`.
- **A rule can fire and an Active Response command can execute successfully, and the actual remediation can still fail** — worth checking the command's own log output (`active-responses.log`), not just whether Wazuh says it ran.

## Next steps

- [ ] Write a custom PowerShell Active Response script to replace the bundled `netsh.exe` binary (parse alert JSON directly, issue `netsh advfirewall` command myself)
- [ ] Test the Linux equivalent (`firewall-drop.sh`) to confirm whether the JSON-parsing bug is Windows-binary-specific
- [ ] Map existing custom rules to MITRE ATT&CK technique IDs
- [ ] Add Suricata/Zeek for network-layer detection alongside host-based FIM/log detections

## Disclaimer

All IPs, bucket names, account IDs, and keys in this repo are placeholders.
This is a personal lab environment, not a production deployment.
