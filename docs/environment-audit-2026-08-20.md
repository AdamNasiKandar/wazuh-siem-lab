# Environment Audit — 2026-08-20

Snapshot of the current Wazuh + AWS environment state, issues found, and
fixes needed before starting the S3/IAM hardening project.

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

## Housekeeping note

Real AWS account ID (`454805669984`) and full IAM ARNs appeared in
plaintext across pasted terminal output during this audit. Not
individually secret, but should be swapped for `<account-id>` /
`<arn>` placeholders before any of this goes into the repo, a
screenshot, or a write-up.

---

## Next steps (in order)

- [ ] Fix container AWS credentials (Issue 1)
- [ ] Update IAM policy with missing CloudTrail/EC2 read permissions (Issue 2)
- [ ] Rename agents 004 and 006 to meaningful names
- [ ] Remove or replace tutorial rule 100001
- [ ] Enable S3 bucket versioning (prerequisite for Object Lock project)
- [ ] Begin S3 Object Lock / hardening project
