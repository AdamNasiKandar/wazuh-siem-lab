# Environment Audit — 2026-08-20

Snapshot of the current Wazuh + AWS environment state, issues found, and
fixes needed before starting the S3/IAM hardening project.

---

## 0. Scope — Amazon services in use

1. **EC2** — hosts the Wazuh manager/indexer/dashboard stack via Docker Compose
2. **CloudTrail** — logs AWS account activity (API calls, IAM changes, etc.)
3. **S3** — stores CloudTrail logs, ingested by Wazuh's aws-s3 wodle
4. **IAM** — scoped users/policies for Wazuh's log-reading access and audit checks

---

## 1. Docker / Manager

3 containers running: `dashboard`, `indexer`, `manager`.

⚠️ **Containers showed `Up 3 minutes`** at time of audit — meaning a
recent restart or recreation happened. This matters because it's the
likely cause of Issue #1 below (container writable-layer data doesn't
survive recreation).

## 2. Agents enrolled

| ID | Name | IP | Status | Notes |
|---|---|---|---|---|
| 000 | wazuh.manager | 127.0.0.1 | Active/Local | The manager itself, not a real endpoint |
| 004 | localhost.localdomain | any | Active | Likely the RHEL VM — generic default hostname |
| 006 | MSI | any | Active | Likely the Windows PC — installer placeholder name |

**Action item:** rename both agents to something identifiable
(`rhel-vm`, `windows-laptop`) so the dashboard/`agent_control -l` output
is actually readable going forward.

## 3. Custom rules (`local_rules.xml`)

| Rule ID | Purpose | Status |
|---|---|---|
| 100001 | Tutorial example — hardcoded to IP `1.1.1.1` | Not functional / cruft, candidate for deletion |
| 100100 | AlienVault reputation list match | Working (per prior troubleshooting doc) |
| 100102 | Frequency-based "multiple web requests from same source" (`frequency=2`, `timeframe=60`) | **Working** — this is the rule Active Response's `netsh` command is bound to (`<rules_id>100102</rules_id>`) |

---

## Issues found

### Issue 1 — Wodle broken: `NoCredentialsError`

```
botocore.exceptions.NoCredentialsError: Unable to locate credentials
```

**Root cause:** `/root/.aws/credentials` inside the manager container
lives in the container's **writable layer**, not a named Docker volume.
It does not survive container recreation — only a plain restart. The
`Up 3 minutes` status above strongly suggests the container was
recreated, wiping this file.

**Fix — run inside the manager container (not the EC2 host):**
```bash
docker exec -it single-node-wazuh.manager-1 bash
mkdir -p /root/.aws

cat > /root/.aws/credentials << 'EOF'
[default]
aws_access_key_id = <access-key-id>
aws_secret_access_key = <secret-access-key>
EOF

cat > /root/.aws/config << 'EOF'
[default]
region = ap-southeast-5
EOF
```

**Note:** this is a *separate* credential file from the `aws configure`
done on the EC2 host as `ec2-user`. That one is for running `aws` CLI
commands over SSH; this one is what the `aws-s3` wodle inside the
container uses to pull CloudTrail logs. Both are needed, independently.

### Issue 2 — IAM audit user missing permissions

`wazuh-cloudtrail-reader` currently denied on:
- `cloudtrail:DescribeTrails`
- `cloudtrail:GetTrailStatus`
- `ec2:DescribeSecurityGroups`

**Fix — add as a new statement (same policy or a separate one):**
```json
{
  "Effect": "Allow",
  "Action": [
    "cloudtrail:DescribeTrails",
    "cloudtrail:GetTrailStatus",
    "ec2:DescribeSecurityGroups"
  ],
  "Resource": "*"
}
```
`"*"` is expected here — these specific read-only calls don't support
resource-level restriction, unlike the S3 actions below.

### Finding — `GetBucketVersioning` returned no output

Not a permissions error — this strongly suggests **versioning is not yet
enabled** on the CloudTrail bucket. Relevant later: **S3 Object Lock
requires versioning to be enabled first**, so this needs addressing
before the Object Lock project can proceed.

### Issue 3 — Phantom duplicate agent on restart ("MSI" reappearing)

**Symptom:** After renaming the Windows agent from its default `MSI` to
`AdamsLaptop` (via remove + re-add through `manage_agents`), a restart of
either the laptop or the EC2 manager caused a **new** agent entry named
`MSI` to appear in the dashboard, while `AdamsLaptop` was left behind,
disconnected.

**Root cause:** Wazuh agent enrollment is **enabled by default since
4.3+**, even with no explicit `<enrollment>` block present in
`ossec.conf`. When the agent's connection was disrupted (service
restart, lost `client.keys`, etc.), it silently fell back to
auto-enrolling itself fresh — using the machine's actual Windows
hostname as the agent name. Confirmed via:
```powershell
hostname
```
which returned `MSI` — the laptop's hostname, not leftover installer
naming as originally assumed.

**First fix attempt broke the agent entirely:**
```xml
<enrollment>
  <enabled>no</enabled>
</enrollment>
```
placed as a **top-level element directly under `<ossec_config>`**.
Result, visible in `ossec.log` after the next restart:

**Linux-side follow-up:** the same phantom-agent pattern occurred on the
RHEL VM, but with a different underlying mechanism than the Windows fix
above.

- `manage_agents`'s "extract key" (option `E`) generates a key intended
  for **manual import** into `client.keys` — it is not compatible with
  `agent-auth`, which performs its own live SSL-based enrollment against
  the manager's `authd` service (port 1515) and ignores a `-k` value
  passed to it as if it were that manual key.
- Without an explicit name, `agent-auth` defaults to the **actual
  machine hostname** — and the VM's hostname was literally `localhost`,
  so every enrollment attempt registered as `localhost.localdomain`,
  colliding with the previous entry of the same name still known to the
  manager (`ERROR: Duplicate agent name`).

---

## What's already working / confirmed

- ✅ IAM user auth via access keys (`aws configure` on EC2 host)
- ✅ S3 bucket policy read — confirmed CloudTrail's standard
  `AWSCloudTrailAclCheck` / `AWSCloudTrailWrite` statements are in place,
  scoped correctly to the trail's ARN
- ✅ Public access block — all four settings (`BlockPublicAcls`,
  `IgnorePublicAcls`, `BlockPublicPolicy`, `RestrictPublicBuckets`) are
  `true` — bucket is not publicly accessible
- ✅ Object Lock check ran successfully (returned the expected
  "not configured" error, not an access error) — confirms permissions
  fix worked, and confirms Object Lock genuinely isn't set up yet
  (expected, since it's the next project)

---

## Resolutions

All issues found in this audit have been fixed and verified as of
2026-08-20.

### Issue 1 — Wodle `NoCredentialsError` → Resolved

Recreated `/root/.aws/credentials` and `/root/.aws/config` inside the
manager container (writable-layer files that don't survive container
recreation). Re-ran the wodle manually with `--debug 2` — traceback no
longer appears. Confirmed via dashboard: **Modules → Amazon AWS**
showing recent-timestamp events again.

### Issue 2 — IAM audit user missing permissions → Resolved

Added `cloudtrail:DescribeTrails`, `cloudtrail:GetTrailStatus`, and
`ec2:DescribeSecurityGroups` (Resource: `*`, since these calls don't
support resource-level restriction) to `wazuh-cloudtrail-reader`'s
policy. Re-ran the previously-failing commands — all return real JSON
output instead of `AccessDeniedException`.

### Issue 3 - Phantom duplicate agent on restart ("MSI" reappearing) → Resolved

The agent had actually reconnected successfully under the renamed
identity right before this — the restart to apply the enrollment
setting is what broke it, not the rename itself. Worth noting since the
log timeline made this easy to misread as "the rename didn't work,"
when it had, in fact, worked until the very next restart.

**Actual fix:** `<enrollment>` must be nested **inside `<client>`**, not
as a sibling of it:
```xml
<client>
  <server>
    <address><manager-ip></address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
  <config-profile>windows, windows11</config-profile>
  <notify_time>10</notify_time>
  <time-reconnect>60</time-reconnect>
  <auto_restart>yes</auto_restart>
  <crypto_method>aes</crypto_method>
  <enrollment>
    <enabled>no</enabled>
  </enrollment>
</client>
```
After correcting the nesting and restarting the service, the agent
reconnected as `AdamsLaptop` with no configuration error and no new
phantom agent created.

**Verification:** confirmed via `ossec.log` showing `Connected to the
server` / `Agent is now online` with no subsequent config error, and via
`agent_control -l` on the manager showing only `AdamsLaptop` active —
no new `MSI` entry.

**Takeaway:** XML schema nesting in Wazuh's config isn't always
intuitive from the docs alone — the same tag name can be valid in one
location and rejected outright in another, and the resulting error
(`No client configured. Exiting`) doesn't obviously point back to "wrong
nesting level" as the cause.

**Actual fix for Linux VM (RHEL 8.10):** force the agent name explicitly at enrollment time, rather than
relying on hostname defaulting:
```bash
sudo /var/ossec/bin/agent-auth -m <manager-ip> -A RHEL-8.10-VM
```
Confirmed working — manager and dashboard now show both `AdamsLaptop`
and `RHEL-8.10-VM` as distinct, active agents.

**Longer-term fix considered but not required:** the VM's hostname
itself is still `localhost`, which would cause the same collision again
on any future fresh auto-enrollment. Renaming it via `hostnamectl
set-hostname` would resolve this at the source, independent of the
Wazuh-specific fix above.

### Finding — Bucket versioning not enabled → Resolved

Enabled via:
```bash
aws s3api put-bucket-versioning --bucket <bucket-name> --versioning-configuration Status=Enabled
```
Confirmed:
```json
{ "Status": "Enabled" }
```

**Permission handling — just-in-time grant/revoke:**
`s3:PutBucketVersioning` was added to the IAM policy only long enough to
run this one command, then removed immediately after, restoring
`wazuh-cloudtrail-reader` to read-only. Verified the revoke actually
took effect by re-attempting `put-bucket-versioning` and confirming it
now fails with `AccessDenied`.

This was a deliberate choice: rather than leaving standing write access
on an otherwise read-only audit identity "in case it's needed again,"
the permission was granted only for the specific one-off action it was
needed for. This mirrors the principle behind just-in-time / temporary
elevated access in production environments, done manually here since
there's no automation tooling in this lab.

**Decision on read/write separation:** considering splitting this into
two IAM users (one strictly read-only, one for write/hardening
actions) as the more production-representative pattern.

### Root user access key — checked, not an issue

Verified via `aws sts get-caller-identity` that the CLI is authenticating
as the IAM user `wazuh-cloudtrail-reader`, not the root account. No
action needed.

---

## Housekeeping note

Real AWS account ID and full IAM ARNs appeared in
plaintext across pasted terminal output during this audit. Not
individually secret, but should be swapped for `<account-id>` /
`<arn>` placeholders before any of this goes into the repo, a
screenshot, or a write-up.

Elastic IP setup, so agents are not required to reconfigure every time AWS server is stopped/restarted

---

## Next steps (in order)

- [x] Fix container AWS credentials (Issue 1)
- [x] Update IAM policy with missing CloudTrail/EC2 read permissions (Issue 2)
- [x] Enable S3 bucket versioning (prerequisite for Object Lock project)
- [ ] Rename agents 004 and 006 to meaningful names
- [ ] Remove or replace tutorial rule 100001
- [ ] Begin S3 Object Lock / hardening project
