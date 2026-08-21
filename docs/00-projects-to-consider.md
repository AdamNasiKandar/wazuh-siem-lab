Small-to-medium projects 

- **boto3 CSPM-style script** — automate the manual bucket checks (policy, public access, versioning, encryption) into a reusable script with pass/fail output
- **CloudTrail log file validation** — enable the SHA-256 digest chain, write a script to verify the log chain hasn't been tampered with
- **Write-only IAM user** — the one still on the back burner, separating read (wazuh-cloudtrail-reader) from write/hardening actions
- **MITRE ATT&CK mapping** — tag your existing rules (100100, 100102) with technique IDs and document it

---

Larger, capstone-style projects

- **S3 Object Lock** — fully unblocked now (versioning enabled, permissions sorted); makes CloudTrail logs genuinely tamper-proof even against an attacker with delete permissions
- **Full least-privilege IAM refactor** — use CloudTrail's own logs to see what your IAM users actually call over time, then rewrite policies down to exactly that
- **EC2 attack-surface reduction** — security group audit, IMDSv2 enforcement, closing port 22 in favor of Session Manager
- **Cross-service capstone** — simulate a suspicious action with a throwaway IAM user, trace it through the full chain (CloudTrail → S3 → Wazuh ingestion → your custom rule alerting)
