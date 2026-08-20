# Adding a Wazuh agent

Two different flows depending on OS. Linux typically auto-enrolls using
just the manager's IP. Windows (via the MSI installer's GUI) usually needs
a manually generated authentication key instead.

---

## Linux (auto-enrollment — simplest)

Works because the install itself handles enrollment over the network,
using `WAZUH_MANAGER` as the target.

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

Install with the manager address set inline:

```bash
sudo WAZUH_MANAGER="<manager-ip>" yum install -y wazuh-agent
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
```

Confirm:

```bash
systemctl status wazuh-agent --no-pager -l
tail -20 /var/ossec/logs/ossec.log
```

Look for a line confirming successful connection to the manager. No
manual key needed — enrollment happens automatically as part of the
install/first start.

---

## Windows (MSI installer — needs a manual key)

The graphical MSI installer's config UI (Manager IP + Authentication key
fields) expects the key to already exist — it doesn't auto-enroll the way
the Linux `yum`/`apt` install does.

### 1. Install the agent

Download the MSI from Wazuh's package server and run it (as
Administrator):

```powershell
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-<version>.msi" -OutFile "wazuh-agent.msi"
msiexec.exe /i .\wazuh-agent.msi WAZUH_MANAGER="<manager-ip>"
```

Run it **without** `/qn` or `/q` (silent flags) the first time — if
something's wrong (corrupted download, missing prerequisite), the
graphical installer shows a visible error dialog. Silent mode fails
quietly with no indication anything went wrong.

If you get "This installation package could not be opened," the MSI file
itself is corrupted or incomplete — redownload it fresh and check the
file size looks reasonable (several MB, not a few KB) before retrying:

```powershell
dir .\wazuh-agent.msi
```

### 2. Generate an authentication key on the manager

SSH into the manager (or, if running Wazuh via Docker, exec into the
manager container):

```bash
sudo docker exec -it <manager-container-name> /var/ossec/bin/manage_agents
```

In the interactive menu:

- Press **`A`** — Add a new agent
- Enter a name for the agent (e.g. `windows-laptop`)
- Enter an IP, or type `any` if the agent's IP is dynamic/unknown
- Confirm the addition

Then, still in the same menu:

- Press **`E`** — Extract the key for an agent
- Select the agent you just added by its ID number

This prints a long key string — copy it in full.

### 3. Enter the key in the Windows agent

Open the **Wazuh Agent** app (installed alongside the MSI, usually
findable via Start menu search, or `C:\Program Files (x86)\ossec-agent\win32ui.exe`).

- **Manager IP:** the manager's public IP
- **Authentication key:** paste the key generated in step 2
- Click **Save**

### 4. Start the service

```powershell
Get-Service | Where-Object {$_.Name -like "*wazuh*"}
```

If the service exists but isn't running:

```powershell
Start-Service WazuhSvc
```

(Note: the actual service name may differ slightly by version — confirm
the exact name from the `Get-Service` output above rather than assuming
`wazuhsvc`.)

Or from the Wazuh Agent app itself: **Manage → Start**.

### 5. Confirm enrollment

In the Wazuh dashboard: **Agents** — the new Windows machine should
appear, showing as **Active** within a minute or two.

---

## Troubleshooting

**MSI install returns no visible error, but no service was created**
Almost always means it was run in silent mode (`/q` or `/qn`) and failed
quietly. Rerun without those flags to see the actual installer UI and any
error dialog it shows.

**"Cannot find any service with service name 'wazuhsvc'"**
Check the actual service name first — don't assume it:
```powershell
Get-Service | Where-Object {$_.Name -like "*wazuh*"}
```

**Agent shows "Auth key not imported" / "Require import of authentication key"**
The manager IP alone isn't enough for this installer flow — you need the
key from `manage_agents` on the manager side (see step 2 above).

**Agent installed and key entered, but still not showing as Active in the dashboard**
Check basic network reachability from the Windows machine to the manager,
on ports 1514 and 1515 — same considerations as any agent: firewall rules
on the manager side, and (if the manager is cloud-hosted) security group
rules must allow inbound traffic from the agent's IP.
