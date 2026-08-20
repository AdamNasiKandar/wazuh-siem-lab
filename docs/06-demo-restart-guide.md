# Wazuh SIEM demo environment — setup and restart guide

Two pieces make up this environment:

- **Wazuh dashboard/manager/indexer** — runs in Docker on the Windows host
- **Wazuh agent** — runs inside a RHEL VM, reports back to the manager

---

## Part 1 — First-time setup

### 1.1 Prerequisites

- Docker Desktop installed and working (WSL2 backend enabled — see
  troubleshooting below if virtualization isn't detected)
- A VM (this guide used RHEL 8.10) reachable on the same network as the
  Docker host
- VMware Workstation, or any hypervisor, for the agent VM

### 1.2 Stand up the Wazuh stack in Docker

```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.x.x
cd wazuh-docker/single-node
docker compose up -d
```

Check that all three containers came up:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```

You should see something like:

```
single-node-wazuh.dashboard-1
single-node-wazuh.manager-1
single-node-wazuh.indexer-1
```

First boot takes **1–3 minutes** — the indexer builds indices, the manager
connects, and only then does the dashboard become reachable. Hitting the
dashboard URL too early gives an empty/broken page — that's expected, not a
failure. If you want to confirm it's genuinely progressing rather than stuck:

```bash
docker logs <dashboard-container-name> --tail 20
```

Look for a line like `Server running at https://0.0.0.0:5601` — that
confirms the dashboard is actually up.

### 1.3 Open the dashboard

```
https://localhost
```

Expect a self-signed certificate warning (this setup uses default certs) —
click through it (Advanced → Accept the Risk and Continue).

Default login is `admin` / whatever password is set in the compose file —
check if unsure:

```bash
grep -i password ~/wazuh-docker/single-node/docker-compose.yml
```

### 1.4 Get the Docker host's LAN IP

The agent VM needs to reach the manager over the network — `localhost`
won't work from inside the VM. On the Windows host:

```powershell
ipconfig
```

Note the IPv4 address of your active network adapter. This is
`<your-host-ip>` in the steps below.

### 1.5 Install the Wazuh agent on the VM

Add the Wazuh repo (RHEL/CentOS-based):

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
```

Install with the manager address set inline, so the agent knows where to
report:

```bash
sudo WAZUH_MANAGER="<your-host-ip>" yum install -y wazuh-agent
```

Enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
systemctl status wazuh-agent --no-pager -l
```

If the service fails immediately with something like:

```
/usr/bin/env: '/var/ossec/bin/wazuh-control': No such file or directory
```

this usually means the systemd unit tried to start before the package
finished laying down files (e.g. the failed status message is just stale).
Confirm the binary now exists and restart:

```bash
ls -la /var/ossec/bin/
sudo systemctl restart wazuh-agent
```

### 1.6 Confirm the agent is connected

```bash
tail -20 /var/ossec/logs/ossec.log
```

Look for a line indicating a successful connection to the manager. Then
check the dashboard — under **Agents**, the VM should show as **Active**
within a minute or two.

### 1.7 Generate a demo alert (failed SSH logins)

Confirm SSH is running on the VM:

```bash
sudo systemctl enable --now sshd
```

From another machine (or the VM itself), attempt to SSH in with the wrong
password 3–6 times in a row:

```bash
ssh <user>@<vm-ip>
```

In the dashboard, go to **Modules → Security events** (or **Threat
hunting**, depending on version) and filter by the agent. You should see
alerts like *"Multiple authentication failures"* appear within a minute.

If nothing shows up, check the raw log and the agent's own processing log:

```bash
sudo tail -30 /var/log/secure
sudo tail -30 /var/ossec/logs/ossec.log
```

---

## Part 2 — Restarting the environment (after shutdown)

Once the initial setup is done, restarting is much faster — no
re-initialization, no cert regeneration.

### 2.1 Start the Docker stack

```bash
cd ~/wazuh-docker/single-node
docker compose up -d
docker ps
```

Should be reachable at `https://localhost` within 20–30 seconds (much
faster than first boot, since data/certs already exist).

### 2.2 Start the VM

Power it on normally through VMware. The Wazuh agent is enabled to
autostart, but worth a quick check once it's booted:

```bash
systemctl status wazuh-agent --no-pager
```

If it's not running:

```bash
sudo systemctl start wazuh-agent
```

### 2.3 Order matters

Start **Docker/dashboard first, then the VM** — that way the agent finds
the manager already listening when it starts, instead of retrying for a
bit before connecting.

### 2.4 Watch out for IP changes

The agent's config has the manager's IP baked in
(`/var/ossec/etc/ossec.conf`, `<address>` field under `<client>`). If your
Windows host's IP changes between sessions (common with DHCP), the agent
won't reconnect until that field is updated to match the new host IP.

For a demo you want to reliably repeat, either:

- Set a static IP on the host's network adapter, or
- Check `ipconfig` before each demo and update `ossec.conf` if it's
  changed, then `sudo systemctl restart wazuh-agent`

---

## Troubleshooting notes

**Docker Desktop: "Virtualization support not detected"**
Usually a WSL2/Hyper-V feature or conflict issue on the Windows host, not
an actual lack of hardware virtualization (especially if VMware already
works fine). Check:

```powershell
Get-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux
```

Both should show `Enabled`. If not, enable them and reboot:

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

A full host reboot after enabling these is often what actually fixes it.

**"No certificate for the indexer node ... in wazuh-install-files.tar"**
(Applies to a manual/VM-based multi-node install, not the Docker
single-node setup.) Means `config.yml` was edited after certificates were
generated, or a node name mismatch exists. Regenerate certs from the
current config:

```bash
./wazuh-certs-tool.sh -A
tar -tvf wazuh-install-files.tar | grep <node-name>
```

Confirm the node name appears in the tarball before retrying the install.

### Update — Elastic IP attached

An Elastic IP has since been attached to the EC2 instance, resolving the
IP-change problem described above. The public IP is now static across
stop/start cycles, so agents no longer need `ossec.conf`'s `<address>`
field updated after a restart — confirmed both the RHEL VM and PC agent
still show **Active** in the dashboard without any reconfiguration.

**Note:** Elastic IPs are free while attached to a running instance, but
AWS charges an hourly fee if one is allocated but left unattached (e.g.
instance stopped for an extended period without being terminated) —
worth remembering if this environment is ever paused long-term.
