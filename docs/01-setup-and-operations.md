# Setup & Operations

Everything needed to stand up, operate, and restart this environment:
AWS instance sizing, the Docker-based Wazuh stack, CloudTrail
integration, enrolling agents (Linux + Windows), and a day-to-day
command reference.

**Current architecture:** Wazuh manager/indexer/dashboard run via Docker
Compose on an AWS EC2 instance (Elastic IP attached, so the public IP is
now static). Two agents report in: a Windows laptop and a RHEL 8.10 VM.

---

## Part 1 — Launching a properly-sized EC2 instance

### Lessons learned the hard way

- **1GB RAM (`t2.micro`/`t3.micro`) is not enough.** The indexer's JVM
  alone tries to reserve ~1GB heap by default — on a 1GB total instance
  this fails immediately with `insufficient memory for the Java Runtime
  Environment`.
- **2GB RAM (`t3.small`) is borderline.** It can run the stack at rest,
  but under load (e.g. mid crash-loop-retry, or right after startup) it
  can still hit OOM kills severe enough to freeze SSH itself, not just
  the container.
- **~4GB RAM is comfortable.** Something like `c7i-flex.large` or
  `t3.medium` (if available in your region/account) gives real headroom
  with no tuning tricks needed.
- **20GB disk was not enough once fully initialized** — three service
  images plus data volumes filled it completely. Use 30GB+.
- **New/unverified AWS accounts may be capped to only the smallest
  instance sizes** ("not eligible under the Free Plan" tooltip on larger
  types), independent of region. Check **Service Quotas → Running
  On-Demand Standard instances** for a self-service increase request, or
  just pick from what's actually available to your account.
- **Newer AWS regions may not offer every instance size** even within a
  supported family. If your target size doesn't appear, check region
  availability before assuming it's an account issue.

### Launch checklist

- **Instance type:** something with ~4GB RAM (see above)
- **Storage:** 30GB+
- **Elastic IP:** attach one — without it, the public IP changes on
  every Stop/Start, breaking agent reconnection (see Part 5)
- **Security group inbound rules:**

  | Port | Purpose | Source |
  |---|---|---|
  | 22 | SSH | My IP |
  | 443 | Dashboard | My IP |
  | 1514-1515 | Agent enrollment/events | Agent source IP(s), or 0.0.0.0/0 if agents are unpredictable |

**Note:** "My IP" is resolved once, at the time you create the rule — if
your home/ISP IP changes later, SSH will silently fail (looks identical
to the instance itself being unreachable). If SSH suddenly stops working
with no other symptoms, check your current IP first:
```bash
curl ifconfig.me
```
and compare it against the security group rule before assuming something
is wrong with the instance.

---

## Part 2 — Installing Docker + the Wazuh stack

```bash
sudo dnf install -y docker git findutils curl wget nano
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
# log out/in for group change to apply, or prefix commands with sudo
```

Docker Compose plugin is often missing on a fresh Amazon Linux instance —
install the standalone binary instead:
```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```
Use `docker-compose` (hyphenated) throughout, not `docker compose`.

Set `vm.max_map_count` **before** starting anything — OpenSearch requires
this, and it's not set high enough by default:
```bash
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

Clone (pin to an actual version tag, not a literal placeholder):
```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.7
cd wazuh-docker/single-node   # NOT the repo root — that's the full HA cluster variant
```

Generate the SSL certs — required before first start, or containers fail
to mount cert files:
```bash
sudo docker-compose -f generate-indexer-certs.yml run --rm generator
```
If this fails with `find: command not found`, install `findutils` first
and retry.

Start the stack:
```bash
sudo docker-compose up -d
```

Confirm all three containers are up:
```bash
sudo docker ps --format "table {{.Names}}\t{{.Status}}"
```
Expect:
```
single-node-wazuh.dashboard-1
single-node-wazuh.manager-1
single-node-wazuh.indexer-1
```
First boot takes **1–3 minutes** — the indexer builds indices, the
manager connects, and only then does the dashboard become reachable.
Hitting the dashboard URL too early gives an empty/broken page — that's
expected, not a failure.

Dashboard: `https://<elastic-ip>` — expect a self-signed cert warning
(default certs), click through it. Default login is `admin` / whatever
password is set in the compose file:
```bash
grep -i password ~/wazuh-docker/single-node/docker-compose.yml
```

---

## Part 3 — CloudTrail integration

Lets Wazuh monitor your AWS account's own activity (API calls, IAM
changes, security group edits, instance launches) — not just what's
happening inside a monitored host.

### 3.1 Create a CloudTrail trail

- AWS Console → **CloudTrail → Create trail**
- Storage: new S3 bucket, note the exact bucket name
- Management events: keep enabled (Read + Write)
- KMS encryption: optional. If enabled, extra IAM permissions are needed
  (see 3.2) plus a KMS key alias.

### 3.2 Create an IAM user with read + decrypt access

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::<bucket-name>",
        "arn:aws:s3:::<bucket-name>/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["kms:Decrypt"],
      "Resource": "*"
    }
  ]
}
```
Drop the KMS statement entirely if encryption wasn't enabled. If using
KMS, also check the **KMS key's own key policy** (separate from the IAM
policy above) explicitly grants this IAM user `kms:Decrypt` — a
commonly missed step, since KMS keys have their own resource-based
policy in addition to IAM's identity-based policy.

Generate access keys for this user (use case: "Application running on an
AWS compute service") and save both values.

### 3.3 Install boto3 and configure credentials on the manager

```bash
sudo docker exec -it <manager-container-name> bash
```
Inside the container:
```bash
dnf install -y python3-pip   # Amazon Linux base image
python3 -m pip install boto3

mkdir -p /root/.aws
cat > /root/.aws/credentials << 'EOF'
[default]
aws_access_key_id = <access-key-id>
aws_secret_access_key = <secret-access-key>
EOF

cat > /root/.aws/config << 'EOF'
[default]
region = <bucket-region>
EOF
```
**The region matters even if you never explicitly picked one** — boto3
defaults to `us-east-1` if unset, and fails with
`IllegalLocationConstraintException` against a bucket in any other
region.

### 3.4 Add the wodle to ossec.conf

```bash
nano /var/ossec/etc/ossec.conf
```
Add before the final `</ossec_config>`:
```xml
<wodle name="aws-s3">
  <disabled>no</disabled>
  <interval>10m</interval>
  <run_on_start>yes</run_on_start>
  <skip_on_error>yes</skip_on_error>
  <bucket type="cloudtrail">
    <name><bucket-name></name>
  </bucket>
</wodle>
```
**Double-check the closing tag is `</wodle>`, not `<wodle>`.** A missing
`/` here silently breaks the entire config — every daemon fails to start
(`wazuh-db` in particular, with an unhelpful "line 0" error) because the
XML can't be parsed at all.

Restart just the manager:
```bash
sudo docker-compose restart wazuh.manager
```

### 3.5 Verify

```bash
sudo docker exec -it <manager-container-name> /var/ossec/wodles/aws/aws-s3 \
  --bucket <bucket-name> --type cloudtrail --debug 2
```
`--debug 2` surfaces the real underlying error (region mismatch, decrypt
failure, credentials missing, etc.) far more clearly than the generic
"Unknown error / exit code 1" the scheduled run logs.

Check the dashboard: **Cloud Security → Amazon AWS** (or Threat
Intelligence, depending on version). Widen the time range if nothing
shows up.

---

## Part 4 — Adding a Wazuh agent

Two different flows depending on OS.

### Linux (auto-enrollment — simplest)

```bash
cat > /etc/yum.repos.d/wazuh.repo << 'EOF'
[wazuh]
gpgcheck=1
gpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH
enabled=1
name=EL-$releasever - Wazuh
baseurl=https://packages.wazuh.com/4.x/yum/
protocol=https
EOF

sudo WAZUH_MANAGER="<manager-ip>" yum install -y wazuh-agent
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent

systemctl status wazuh-agent --no-pager -l
tail -20 /var/ossec/logs/ossec.log
```
Look for a line confirming successful connection to the manager.

**Explicitly naming the agent (recommended):** rather than relying on
hostname-based auto-enrollment defaults — see the agent-naming gotchas
in `docs/03-environment-audit.md` — use `agent-auth` directly with an
explicit name:
```bash
sudo /var/ossec/bin/agent-auth -m <manager-ip> -A <chosen-agent-name>
sudo systemctl restart wazuh-agent
```

### Windows (MSI installer — needs a manual key)

The graphical MSI installer's config UI expects an authentication key to
already exist — it doesn't auto-enroll the way the Linux install does.

**1. Install the agent** (as Administrator, non-silent first — silent
mode fails quietly on error):
```powershell
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-<version>.msi" -OutFile "wazuh-agent.msi"
msiexec.exe /i .\wazuh-agent.msi WAZUH_MANAGER="<manager-ip>"
```
If you get "This installation package could not be opened," the MSI is
corrupted/incomplete — redownload and check file size looks reasonable.

**2. Generate an authentication key on the manager:**
```bash
sudo docker exec -it <manager-container-name> /var/ossec/bin/manage_agents
```
- Press `A` — add a new agent, name it, IP or `any`
- Press `E` — extract the key for that agent, copy it in full

**3. Enter the key in the Windows Agent app** (Start menu, or
`C:\Program Files (x86)\ossec-agent\win32ui.exe`): Manager IP + paste the
key → Save.

**4. Start the service:**
```powershell
Get-Service | Where-Object {$_.Name -like "*wazuh*"}
Start-Service WazuhSvc   # confirm exact name from the line above first
```

**5. Confirm:** dashboard → Agents → new machine shows Active within a
minute or two.

### Agent troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| MSI install shows no error, but no service created | Ran in silent mode (`/q`/`/qn`), failed quietly | Rerun without silent flags to see the actual installer UI/error |
| "Cannot find any service with service name 'wazuhsvc'" | Service name varies by version | `Get-Service \| Where-Object {$_.Name -like "*wazuh*"}` first |
| "Auth key not imported" | Manager IP alone isn't enough for MSI flow | Need the key from `manage_agents` (step 2 above) |
| Key entered but agent not Active in dashboard | Network/firewall/security-group blocking ports 1514/1515 | Check reachability + security group inbound rules |
| Service fails immediately: `/usr/bin/env: '/var/ossec/bin/wazuh-control': No such file or directory` (Linux) | systemd unit started before package finished laying down files | `ls -la /var/ossec/bin/` to confirm binary exists, then `sudo systemctl restart wazuh-agent` |
| Agent re-registers under an unexpected name after a restart | Auto-enrollment defaulting to machine hostname | See the phantom-agent case studies in `docs/03-environment-audit.md`; explicitly name via `agent-auth -A <name>`, and nest any `<enrollment><enabled>no</enabled></enrollment>` block **inside `<client>`**, not as a top-level element |

---

## Part 5 — Day-to-day operation & restarting
### Starting the stack

Since all three services have `restart: always` set in `docker-compose.yml`,
the Wazuh stack **auto-starts** whenever the EC2 instance boots and Docker
comes up — no manual `docker-compose up -d` needed after a stop/start cycle.
Confirm it's actually up before assuming a problem:

```bash
sudo docker ps --format "table {{.Names}}\t{{.Status}}"
```

### Order still matters, just differently now

Even though the stack auto-starts, it still takes **1–3 minutes** to fully
initialize (indexer building indices, manager connecting) before it's ready
to accept agent connections. Start the **EC2 instance first**, then the
agents/VM a few minutes later — agents that connect too early will just
retry until the manager is ready, not fail outright, but starting the
instance first avoids unnecessary retry delay.

**Manual `docker-compose up -d` is only needed if:**
- You explicitly ran `docker-compose down` (stops *and* removes containers —
  `restart: always` has nothing to restart)
- You're setting up the stack fresh for the first time

### IP changes — resolved via Elastic IP

An Elastic IP is now attached to the EC2 instance, so the public IP is
static across stop/start cycles — agents no longer need `ossec.conf`'s
`<address>` field updated after a restart. Confirmed both agents
reconnect without reconfiguration.

**Cost note:** Elastic IPs are free while attached to a running
instance, but AWS charges an hourly fee if one is allocated but left
**unattached** (e.g. instance stopped long-term without being
terminated) — worth remembering if this environment is ever paused for
an extended period.

---

## Command reference

A running reference of commands used repeatedly across setup, debugging,
and day-to-day operation.

### Container management
```bash
sudo docker ps --format "table {{.Names}}\t{{.Status}}"          # running only
sudo docker ps -a --format "table {{.Names}}\t{{.Status}}"       # include stopped
sudo docker logs <container-name> --tail 50
sudo docker logs <container-name> --tail 100 | grep -i <keyword>

sudo docker-compose restart <service-name>     # e.g. wazuh.manager
sudo docker-compose up -d --force-recreate <service-name>
sudo docker-compose down          # stops + removes containers, KEEPS volumes
sudo docker-compose down -v       # also deletes volumes — config/data loss
```
**Always `cd` into the directory containing `docker-compose.yml`** before
running any `docker-compose` command — wrong directory gives `no
configuration file provided: not found`, and there's no systemd unit for
any of this (`systemctl restart wazuh-manager` will not work).

### Entering a container
```bash
sudo docker exec -it <container-name> bash
```
Common container names: `single-node-wazuh.manager-1`,
`single-node-wazuh.indexer-1`, `single-node-wazuh.dashboard-1`

**Git Bash / MINGW64 on Windows only:** paths starting with `/` can get
mangled into a Windows path by MSYS's automatic path conversion. If a
command like `docker exec ... /var/ossec/bin/wazuh-logtest` fails with a
garbled `C:/Program Files/Git/...` error, double the leading slash
(`//var/ossec/...`) or prefix the whole command with
`MSYS_NO_PATHCONV=1`.

### Inside the manager container
```bash
/var/ossec/bin/wazuh-control status
/var/ossec/bin/wazuh-control restart

# Test a raw log line against the live ruleset (shows decoder + rule match)
/var/ossec/bin/wazuh-logtest

# Manually add/remove/extract keys for agents
/var/ossec/bin/manage_agents

# AWS wodle — run manually with debug output
/var/ossec/wodles/aws/aws-s3 --bucket <bucket-name> --type cloudtrail --debug 2
```

### File ownership / permissions (manager container)
```bash
ls -la /var/ossec/etc/

# docker cp writes files as root:root by default — this breaks
# wazuh-db (which expects root:wazuh) with a misleading
# "Error reading XML file ... (line 0)" error
chown wazuh:wazuh /var/ossec/etc/ossec.conf
chmod 640 /var/ossec/etc/ossec.conf
```

### Validating XML config

```bash
python3 -c "import xml.etree.ElementTree as ET; ET.parse('/var/ossec/etc/ossec.conf')"
```
Silent = valid, any output = a real syntax error with line/column
location.

**Caveat:** this only works for single-root XML files like
`ossec.conf`. Wazuh rule files (e.g. `local_rules.xml`) legitimately
contain multiple top-level `<group>` blocks side by side, which Python's
strict parser will reject as "junk after document element" even when the
file is perfectly valid for Wazuh itself. For rule files, validate with
`/var/ossec/bin/wazuh-logtest` or just restart and check `ossec.log`
instead.

### SSH
```bash
chmod 400 your-key.pem
ssh -i your-key.pem ec2-user@<instance-public-ip>
```
If SSH hangs/fails with no other symptoms: check `curl ifconfig.me`
against the security group's SSH rule source IP first.

### AWS instance lifecycle (console-only, no CLI needed)
- **Stop/Start**: EC2 → Instances → Instance state (public IP changes on
  Start unless an Elastic IP is attached)
- **Reboot**: keeps the same public IP, unlike Stop/Start
- **Resize**: must be Stopped first → Actions → Instance settings →
  Change instance type
- **Resize EBS volume**: EC2 → Volumes → Modify Volume → increase size,
  then:
  ```bash
  df -T /
  sudo growpart /dev/nvme0n1 1
  sudo xfs_growfs -d /
  ```

### Cost hygiene
- Tag every resource (EC2 instance, volume, security group, S3 buckets,
  CloudTrail trail, KMS key) with `Project = <project-name>` — tags don't
  cascade automatically
- Billing → Cost allocation tags → activate the tag (up to 24h to appear)
- Billing → Cost Explorer → filter by tag to compare spend
- Stop instances when not actively in use — EC2 compute billing stops
  while stopped; only small EBS storage cost continues

---

## Persistence — what survives what

Wazuh's Docker Compose setup uses **named Docker volumes**
(`wazuh_etc`, etc.), not simple host bind-mounts to the `config/` folder
in the repo. That folder only *seeds* the volume on first creation —
editing files there after the fact has no live effect on an
already-running stack.

| Location | Survives `restart` | Survives `down`/`up` (no `-v`) | Survives `down -v` |
|---|---|---|---|
| `/var/ossec/etc/` (named volume `wazuh_etc`) | Yes | Yes | No |
| `/root/.aws/credentials`, `/root/.aws/config` (container writable layer, not a volume) | Yes | **No** | No |
| Anything edited via `docker cp` into a named-volume path | Yes | Yes (check ownership after) | No |

**Practical impact:** if you ever run `docker-compose up -d
--force-recreate` on the manager, or tear down and rebuild the stack,
`/root/.aws/credentials` and `/root/.aws/config` need to be recreated —
this has already caused a real outage once (see
`docs/03-environment-audit.md`, Issue 1).

---

## Troubleshooting quick reference

| Symptom | Cause | Fix |
|---|---|---|
| `insufficient memory for the Java Runtime Environment`, mmap failure | Instance RAM too small for indexer's JVM heap | Resize instance up, or cap heap via `OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m` (band-aid below ~2GB) |
| Container OOM-killed (exit 137) even with heap capped | Total instance RAM still insufficient | Resize instance up |
| `No space left on device` | Disk too small once images + volumes fill it | Resize EBS volume, then `growpart` + `xfs_growfs` |
| SSH suddenly stops working, no other symptoms | Security group's "My IP" rule is stale | `curl ifconfig.me`, compare to the rule, update if different |
| `IllegalLocationConstraintException` from the wodle | boto3 defaulting to `us-east-1`, bucket is elsewhere | Set `region` in `/root/.aws/config` inside the manager container |
| `wazuh-db did not start correctly`, `Error reading XML file ... (line 0)` | Usually a permissions issue on `ossec.conf`, not XML content | Check ownership matches other files in `/var/ossec/etc/` (`root:wazuh`) — `docker cp` tends to write `root:root` |
| `mismatched tag` XML parse error | Genuine syntax error, e.g. `<wodle>` instead of `</wodle>` | Read exact line/column from the error, check that tag pair |

---

## Appendix — earlier local demo setup (superseded)

Before migrating to the AWS EC2 + Docker setup described above, this
environment was first run locally: **Wazuh dashboard/manager/indexer in
Docker Desktop on a Windows host**, with the RHEL VM running under
VMware Workstation on the same machine. This section is kept for
reference but is **no longer the current setup** — retained in case any
of these notes are useful if a local/offline demo environment is ever
needed again.

### Prerequisites (local setup)
- Docker Desktop with WSL2 backend enabled
- A hypervisor (VMware Workstation used here) for the agent VM

### Getting the Docker host's LAN IP
The agent VM needs to reach the manager over the network —
`localhost` doesn't work from inside the VM:
```powershell
ipconfig
```
Note the IPv4 address of the active adapter as `<host-ip>`.

### Local-setup-specific troubleshooting

**Docker Desktop: "Virtualization support not detected"**
Usually a WSL2/Hyper-V feature/conflict issue, not an actual lack of
hardware virtualization (especially if VMware already works). Check:
```powershell
Get-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux
```
Both should show `Enabled`. If not:
```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```
A full host reboot after enabling these is often what actually fixes it.

**"No certificate for the indexer node ... in wazuh-install-files.tar"**
(Applies to a manual/VM-based multi-node install, not Docker
single-node.) Means `config.yml` was edited after certificates were
generated, or a node name mismatch exists:
```bash
./wazuh-certs-tool.sh -A
tar -tvf wazuh-install-files.tar | grep <node-name>
```

**IP changes between sessions (pre-Elastic-IP)**
Before the Elastic IP was attached (see Part 5), the Windows host's
dynamic IP changing between sessions was a recurring problem — the
agent's `<address>` field would go stale, requiring a manual update to
`ossec.conf` and an agent restart every time. This is what the Elastic
IP fix in Part 5 permanently resolved.
