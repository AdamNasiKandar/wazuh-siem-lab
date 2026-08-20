# Command reference — Wazuh / AWS / Docker

A running reference of commands used repeatedly across setup, debugging,
and day-to-day operation. Grouped by purpose, not chronological order.

---

## AWS — instance lifecycle

```bash
# Check your own current public IP (compare against security group "My IP" rules
# if SSH suddenly stops working for no obvious reason)
curl ifconfig.me
```

Console-only actions (no CLI needed for a single instance):
- **Stop / Start**: EC2 → Instances → select → Instance state → Stop/Start
  (Stop preserves the disk; public IP changes on next Start unless you
  have an Elastic IP)
- **Reboot**: EC2 → Instances → select → Instance state → Reboot
  (keeps the same public IP, unlike Stop/Start)
- **Resize**: must be **Stopped** first → Actions → Instance settings →
  Change instance type → Start again
- **Resize EBS volume**: EC2 → Volumes → select → Actions → Modify Volume
  → increase size, then on the instance:
  ```bash
  df -T /                      # confirm filesystem type
  sudo growpart /dev/nvme0n1 1 # adjust device name as needed
  sudo xfs_growfs -d /
  ```

---

## SSH

```bash
chmod 400 your-key.pem
ssh -i your-key.pem ec2-user@<instance-public-ip>
```

If SSH hangs/fails with no other symptoms: check `curl ifconfig.me`
against the security group's SSH rule source IP first — a changed home IP
is the most common silent cause.

---

## Initial instance setup (Amazon Linux 2023)

```bash
sudo dnf install -y docker git findutils curl wget nano
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
# log out/in for group change to apply, or keep using sudo

# Docker Compose plugin is often missing — install the standalone binary
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
# use `docker-compose` (hyphenated), not `docker compose`, going forward

# Required by OpenSearch — must be set before starting the stack
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

---

## Wazuh Docker stack

```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.7
cd wazuh-docker/single-node   # NOT the repo root — that's the full HA cluster variant

# Required before first start, or containers fail to mount cert files
sudo docker-compose -f generate-indexer-certs.yml run --rm generator

sudo docker-compose up -d
```

**Always `cd` into the directory containing `docker-compose.yml` before
running any `docker-compose` command** — running it from the wrong
directory (e.g. your home dir) gives `no configuration file provided: not
found`, and there's no systemd unit for any of this (`systemctl restart
wazuh-manager` will not work).

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

### Entering a container

```bash
sudo docker exec -it <container-name> bash
```

Common container names in this setup:
`single-node-wazuh.manager-1`, `single-node-wazuh.indexer-1`,
`single-node-wazuh.dashboard-1`

**Note (Git Bash / MINGW64 on Windows only):** paths starting with `/`
can get mangled into a Windows path by MSYS's automatic path conversion.
If a command like `docker exec ... /var/ossec/bin/wazuh-logtest` fails
with a garbled `C:/Program Files/Git/...` error, either double the
leading slash (`//var/ossec/...`) or prefix the whole command with
`MSYS_NO_PATHCONV=1`.

---

## Wazuh — inside the manager container

```bash
# Daemon status
/var/ossec/bin/wazuh-control status
/var/ossec/bin/wazuh-control restart

# Test a raw log line against the live ruleset (shows decoder + rule match)
/var/ossec/bin/wazuh-logtest

# Manually add/remove/extract keys for agents (needed for Windows MSI installs,
# which require a manual auth key rather than auto-enrolling)
/var/ossec/bin/manage_agents

# AWS wodle — run manually with debug output (much more useful than the
# generic "exit code 1 / Unknown error" the scheduled run logs)
/var/ossec/wodles/aws/aws-s3 --bucket <bucket-name> --type cloudtrail --debug 2
```

---

## Wazuh agent — Linux

```bash
# Repo setup (RHEL/CentOS-based)
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

---

## Wazuh agent — Windows (PowerShell, run as Administrator)

```powershell
# Download + install (non-silent first — silent mode fails quietly on error)
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-<version>.msi" -OutFile "wazuh-agent.msi"
msiexec.exe /i .\wazuh-agent.msi WAZUH_MANAGER="<manager-ip>"

# Service check/control
Get-Service | Where-Object {$_.Name -like "*wazuh*"}
Restart-Service -Name wazuh    # confirm exact name from the line above first

# Config file
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"
# (needs -Verb RunAs to save changes: Start-Process notepad "..." -Verb RunAs)

# Active response log — check for command execution attempts/errors
Get-Content "C:\Program Files (x86)\ossec-agent\active-response\active-responses.log" -Tail 20

# Uninstall
msiexec.exe /x .\wazuh-agent.msi
```

---

## FIM (File Integrity Monitoring)

Config file locations by OS:

| OS | Path |
|---|---|
| Linux | `/var/ossec/etc/ossec.conf` |
| Windows | `C:\Program Files (x86)\ossec-agent\ossec.conf` |

```xml
<syscheck>
  <disabled>no</disabled>
  <frequency>300</frequency>            <!-- seconds; lower for testing, raise for production -->
  <directories check_all="yes">/tmp</directories>
</syscheck>
```

Restart after editing:
```bash
sudo systemctl restart wazuh-agent          # Linux
```
```powershell
Restart-Service -Name wazuh                 # Windows
```

---

## AWS CloudTrail integration (inside the manager container)

```bash
python3 -m pip install boto3   # if not already present; pip itself may
                                # need installing first: dnf install -y python3-pip

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
**The region setting is easy to forget and causes a non-obvious error**
(`IllegalLocationConstraintException`) if omitted — boto3 defaults to
`us-east-1` otherwise.

`ossec.conf` wodle block:
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

---

## File ownership / permissions (manager container)

```bash
# Check ownership matches the pattern of neighboring files
ls -la /var/ossec/etc/

# docker cp writes files as root:root by default — this breaks
# wazuh-db (which expects root:wazuh) with a misleading
# "Error reading XML file ... (line 0)" error
chown wazuh:wazuh /var/ossec/etc/ossec.conf
chmod 640 /var/ossec/etc/ossec.conf
```

---

## Validating XML config without guessing

```bash
python3 -c "import xml.etree.ElementTree as ET; ET.parse('/var/ossec/etc/ossec.conf')"
```
Silent = valid. Any output = a real syntax error, with line/column
location.

---

## Persistence — what survives what

| Location | Survives `restart` | Survives `down` / `up` (no `-v`) | Survives `down -v` |
|---|---|---|---|
| `/var/ossec/etc/` (named volume `wazuh_etc`) | Yes | Yes | No |
| `/root/.aws/credentials`, `/root/.aws/config` (container writable layer, not a volume) | Yes | **No** | No |
| Anything edited via `docker cp` into a named-volume path | Yes | Yes (but check ownership after) | No |

Editing the repo's local `config/` folder on the host has **no live
effect** on an already-running stack — those files only seed the named
volumes on first creation.

---

## Cost hygiene

```bash
# Nothing to run — done in the console:
# 1. Tag every resource: Key = Project, Value = <project-name>
#    (EC2 instance, its volume, its security group, S3 buckets, CloudTrail
#    trail, KMS key if used — tags do not cascade automatically)
# 2. Billing → Cost allocation tags → activate the "Project" key
#    (takes up to 24h to appear after first use)
# 3. Billing → Cost Explorer → filter by tag to compare spend per project
```

Stop instances when not actively in use — EC2 compute billing stops
while stopped; only small EBS storage cost continues.
