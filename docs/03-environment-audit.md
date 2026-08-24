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
| 004 | RHEL-8.10-VM | any | Active | RHEL VM |
| 006 | AdamsLaptop | any | Active | Windows PC  |

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
sudo /var/ossec/bin/agent-auth -m <manager-ip> -A rhel-8-10-vm
```
Confirmed working — manager and dashboard now show both `AdamsLaptop`
and `RHEL-8.10-VM` as distinct, active agents.

**Longer-term fix — completed:** the VM's hostname has since been
changed from `localhost` via `hostnamectl set-hostname`, removing the
underlying condition that caused this collision in the first place.
Confirmed independently by the Apache/rule-100102 verification work in
Issue 4, where the VM shows up correctly as `RHEL-8.10-VM` throughout.

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

## Issue 4 — End-to-end verification of rule 100102 (successive connections)

**Goal:** confirm rule `100102` (multiple web requests from the same
source within 60 seconds) fires on genuine attacker→victim traffic —
RHEL VM as attacker, Windows PC's Apache as victim — not just a manually
pasted `wazuh-logtest` line.

This turned into a multi-layered troubleshooting session, each fix
uncovering the next issue underneath. Documented in the order
discovered, since each step's root cause only became clear after
ruling out the previous one.

### 4.1 — NAT hairpinning masked the real source IP

Initial test used `curl` from the RHEL VM (running under VMware
Workstation, NAT-mode adapter) against the Windows PC's Apache. The
resulting Apache log entries showed the **Windows PC's own LAN IP**
as the source, not the VM's.

**Root cause:** the VM's NAT adapter routes outbound traffic through
the host machine's own network stack. Since the "attacker" (VM) and
"victim" (Apache) both ultimately live on the same physical host, the
connection hairpins — by the time it reaches Apache, the OS reports the
host's own IP as the source, not the VM's internal NAT address
(`192.168.240.128`/`192.168.122.1`). `wazuh-logtest` fired correctly on
this data (proving the rule logic itself was sound), but it wasn't
testing genuine attacker-distinct traffic.

**Fix:** switched the VM's network adapter from **NAT** to **Bridged**
in VMware Workstation (Edit → Virtual Network Editor), giving the VM
its own real address directly on the LAN (`192.168.0.201`).

### 4.2 — Bridged networking broke over Wi-Fi (partially)

After switching to bridged mode, the VM received a genuine
`192.168.0.x` address, but `ping` from VM → Windows PC failed (PC → VM
ping worked fine).

**Root cause:** the Windows PC connects to the home network over
**Wi-Fi**, and `Get-NetConnectionProfile` showed the connection
classified as **Public**. Windows Defender Firewall's default Public
profile blocks inbound ICMP Echo Requests — explaining the one-way
failure (Linux's default firewall doesn't block outbound/inbound ICMP
the same way).

**Fix:** added an explicit inbound allow rule rather than reclassifying
the whole network profile:
```powershell
netsh advfirewall firewall add rule name="Allow ICMPv4-In (VM lab)" protocol=icmpv4:8,any dir=in action=allow
```
Ping succeeded both directions afterward.

### 4.3 — Leftover Wazuh server components on the VM

While troubleshooting, an unrelated alert surfaced: repeated
`wazuh-dashboard.service` crashes on the RHEL VM (rule `40704`,
`firedtimes: 79` — clogging the alert log).

**Root cause:** a full Wazuh server install (`wazuh-indexer`,
`wazuh-dashboard`) had been done on the VM at some earlier point,
alongside the intended `wazuh-agent`. Both leftover services were
`enabled` and repeatedly failing to start.

**Fix:**
```bash
sudo systemctl stop wazuh-dashboard wazuh-indexer wazuh-indexer-performance-analyzer
sudo systemctl disable wazuh-dashboard wazuh-indexer wazuh-indexer-performance-analyzer
sudo yum remove -y wazuh-dashboard wazuh-indexer
sudo rm -f /etc/systemd/system/wazuh-dashboard.service   # leftover manual unit file blocking a clean `mask`
sudo systemctl daemon-reload
```
`wazuh-agent` was left untouched and confirmed still healthy afterward
(`systemctl status wazuh-agent` — all 5 expected processes running, no
errors). Noted in passing: the VM also has a PostgreSQL install being
picked up by the log collector (`/var/lib/pgsql/data/log/...`), likely
an unrelated leftover from the same earlier install — not touched, just
flagged as a known oddity on this VM.

### 4.4 — Windows agent's Apache `<localfile>` block had disappeared

With networking and the VM cleaned up, fresh curl traffic still didn't
produce a `100102` alert. Checking the Windows agent's `ossec.conf`
showed **only the default out-of-the-box localfile entries**
(Application/Security/System event channels, active-response log) —
the `<localfile>` block for `C:\Apache24\logs\access_log` (added earlier
during the Active Response project, see
`docs/02-detection-capabilities.md`) was gone entirely.

**Root cause not fully confirmed** — most likely explanation is that a
service/MSI-level operation during the earlier agent rename/re-enrollment
work regenerated `ossec.conf` from defaults rather than preserving the
edited version. Worth treating as a known risk on this agent rather
than a one-off.

**Fix:** re-added the block, restarted, and explicitly re-verified it
persisted through the restart before re-testing:
```xml
<localfile>
  <log_format>syslog</log_format>
  <location>C:\Apache24\logs\access_log</location>
</localfile>
```
```powershell
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.conf" | Select-String -Context 2 "Apache24"
```

**Mitigation adopted:** keep a backup copy of a known-working
`ossec.conf` outside the ossec-agent install folder, to restore from
rather than rebuild from memory if this recurs:
```powershell
Copy-Item "C:\Program Files (x86)\ossec-agent\ossec.conf" "$env:USERPROFILE\Desktop\ossec-working-backup.conf"
```

### 4.5 — Confirmed working, end-to-end

With all four issues above resolved, a fresh curl loop from the VM
(`192.168.0.201`) against the Windows PC's Apache produced a genuine,
correctly-attributed alert:

```
Rule: 100102 (level 10) -> "Multiple web requests from same source
within 60 seconds - possible scanning/successive connection"
agent: AdamsLaptop (007)
srcip: 192.168.0.201
firedtimes: 6
```

Active Response triggered immediately after (`netsh.exe - add`,
rule `657`), and the alert appears correctly in the dashboard under the
Windows agent. This is the first fully genuine (not `wazuh-logtest`
simulated) end-to-end confirmation of this rule and its Active Response
binding.

**Note:** Active Response's known downstream limitation — a bug in the
compiled `netsh.exe` binary, confirmed still present and root-caused in
detail — is tracked separately in `docs/04-known-issues.md`, since it
remains unresolved upstream rather than something fixed as part of this
audit.

---

## Issue 5 — Detection rule silently broken (`if_sid` rule-load failure), corrected

**Context:** while re-verifying the `netsh.exe` Active Response fix in a
later session (after switching networks, which triggered several
unrelated network hiccups — see below), rule `100102` stopped
escalating entirely. Real traffic reached Apache correctly, the base
rule `31108` fired correctly, but `100102` never appeared in any alert
output — dashboard, `alerts.log`, or `wazuh-logtest`.

**Two networking hiccups found and fixed along the way, unrelated to
the actual rule bug:**
- Home network's security group rules needed the same "My IP is
  resolved once" fix already documented for SSH — port 443 (dashboard)
  silently failed after switching from office to home network. Added a
  second scoped rule for the home IP rather than replacing the existing
  one, so both networks continue to work.
- RHEL VM's `ens160` interface showed as `unmanaged` in
  `nmcli device status` after a network switch — no udev rule, no
  `NM_CONTROLLED=no`, no conf.d override found; root cause turned out to
  be simpler than any of that: Networking itself was toggled off at the
  NetworkManager level. Fixed with `nmcli networking on`.

**The actual rule bug — a documentation correction:**

`docs/02-detection-capabilities.md` previously documented switching
from `if_matched_sid` to `if_sid` as the fix for this rule not firing.
**That earlier fix was wrong.** Confirmed definitively via
`wazuh-analysisd`'s own error log after reverting to `if_sid` during
this session:
```
ERROR: Invalid use of frequency/context options. Missing if_matched on rule '100102'.
CRITICAL: Error loading the rules: 'etc/rules/local_rules.xml'.
```
This is unambiguous: `if_sid` combined with `frequency`/`timeframe`
causes the entire rules file to fail loading — not silently skip one
rule. The base rule `31108` kept firing throughout because it comes
from Wazuh's separate, valid default ruleset; `100102` never had a
chance to load at all, from either version tested.

**The real fix** — confirmed via a fresh `wazuh-logtest` session
showing clean two-stage escalation — is `if_matched_sid` referencing a
proper **intermediate rule**, not the raw base decoder rule directly:
```xml
<rule id="100101" level="3">
  <if_sid>31108</if_sid>
  <description>Web request logged (intermediate rule for frequency correlation)</description>
  <group>web,</group>
</rule>

<rule id="100102" level="10" frequency="20" timeframe="3">
  <if_matched_sid>100101</if_matched_sid>
  <same_source_ip />
  <description>20+ web requests from same source within 3 seconds - possible scanning/DoS</description>
  <group>web,successive_connection,</group>
</rule>
```
Also took the opportunity to tune the threshold from the original loose
testing values (`frequency="2" timeframe="60"`, easily triggered by
normal browsing) to a more realistic `frequency="20" timeframe="3"`.

**Confirmed end-to-end** with a 25-request concurrent burst from the
RHEL VM: `100101` → `100102` → Active Response → real firewall block,
all visible in the dashboard within the same second. Corrected in
`docs/02-detection-capabilities.md`.

**Lesson worth keeping in mind:** an earlier "fix" that was never
re-verified against a hard failure signal (like an explicit
`analysisd` load error) can sit undetected for a long time if the
symptom it was chasing looks similar to a different, unrelated problem
— in this case, a rule that "doesn't fire" and a rule that "never
loaded" look identical from the dashboard, but have completely
different causes and fixes.

---

## Next steps (in order)

- [x] Fix container AWS credentials (Issue 1)
- [x] Update IAM policy with missing CloudTrail/EC2 read permissions (Issue 2)
- [x] Enable S3 bucket versioning (prerequisite for Object Lock project)
- [x] Rename agents to meaningful names (AdamsLaptop, RHEL-8.10-VM)
- [x] Remove tutorial rule 100001
- [x] Verify rule 100102 end-to-end with genuine attacker→victim traffic (Issue 4)
- [x] Confirm whether the netsh.exe Active Response bug still applies post-verification — confirmed still present, same failure mode
- [x] Root-cause investigation: ruled out config/version/firewall issues, full config review, matched to open upstream GitHub issue #21812, filed reproduction comment — see `docs/04-known-issues.md`
- [x] Write custom PowerShell Active Response script to replace netsh.exe (parse srcip directly, issue netsh advfirewall command) — see `docs/04-known-issues.md`
- [ ] Begin S3 Object Lock / hardening project
