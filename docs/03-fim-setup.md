# Setting up and testing File Integrity Monitoring (FIM)

FIM watches specified files/directories and alerts on creation,
modification, or deletion. Configured per-agent (or centrally via the
manager), and only takes effect after a restart of the agent service.

Official reference: https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity-monitoring/

---

## 1. Locate the config file

The `<syscheck>` block lives in `ossec.conf`, in a different path per OS:

| OS | Path |
|---|---|
| Linux | `/var/ossec/etc/ossec.conf` |
| Windows | `C:\Program Files (x86)\ossec-agent\ossec.conf` |
| macOS | `/Library/Ossec/etc/ossec.conf` |

## 2. Edit the `<syscheck>` block

**Linux:**
```bash
sudo nano /var/ossec/etc/ossec.conf
```

**Windows (needs Administrator to save):**
```powershell
Start-Process notepad "C:\Program Files (x86)\ossec-agent\ossec.conf" -Verb RunAs
```

Add a `<directories>` line per path you want watched:

```xml
<syscheck>
  <disabled>no</disabled>
  <frequency>43200</frequency>
  <directories check_all="yes">/tmp</directories>
</syscheck>
```

Windows example:
```xml
<directories check_all="yes">C:\Users\<username>\Downloads</directories>
```

Multiple paths can go on one line (comma-separated) or as separate
`<directories>` lines. Wildcards (`*`, `?`) are supported, e.g.
`C:\Users\*\Downloads` to cover every user profile at once.

### What `check_all="yes"` does

Enables every available attribute check on files in that directory, not
just "did it appear/disappear":

- **Size**
- **Permissions**
- **Owner (uid) / Group (gid)**
- **Modification time**
- **Inode** (catches a file being replaced entirely at the same path)
- **Hash (MD5/SHA1/SHA256)** — catches content changes even when size
  stays the same

Without `check_all`, you can instead enable only specific checks, e.g.
`check_size="yes" check_owner="yes"` — less thorough but less noisy and
lower overhead. `check_all="yes"` is the right choice while testing, since
it gives full visibility into what FIM is actually capable of detecting.

### Lowering the scan frequency (for testing)

`<frequency>` is in **seconds**. Default is `43200` (12 hours) — far too
slow for iterative testing. Drop it temporarily:

```xml
<frequency>300</frequency>
```

(5 minutes — reasonable balance of "fast enough to test" vs. not
constantly hashing the filesystem)

**Remember to raise this back** (hourly, or the original 12-hour default)
once you're done testing — frequent full scans cost real CPU/disk I/O and
aren't necessary for genuine ongoing monitoring.

## 3. Restart the agent to apply changes

| OS | Command |
|---|---|
| Linux | `sudo systemctl restart wazuh-agent` |
| Windows | `Restart-Service -Name wazuh` (confirm exact service name first: `Get-Service \| Where-Object {$_.Name -like "*wazuh*"}`) |
| macOS | `/Library/Ossec/bin/wazuh-control restart` |

## 4. Trigger a test event

**Linux (`/tmp` example):**
```bash
echo "test file" > /tmp/fim-test.txt
sleep 5
echo "modified" >> /tmp/fim-test.txt
rm /tmp/fim-test.txt
```

**Windows (Downloads example):**
```powershell
New-Item -Path "C:\Users\$env:USERNAME\Downloads\test-fim.txt" -ItemType File
Add-Content -Path "C:\Users\$env:USERNAME\Downloads\test-fim.txt" -Value "test change"
Remove-Item -Path "C:\Users\$env:USERNAME\Downloads\test-fim.txt"
```

## 5. Check the dashboard

Wait up to your configured `<frequency>` interval (scans are scheduled,
not instant/real-time). Then:

**Modules → Security events** (or a dedicated Integrity Monitoring /
FIM module, depending on version), filtered by the relevant agent.

You should see separate alerts for creation, modification, and deletion —
each carrying the specific attribute(s) that changed (hash, size, mtime,
etc.) if `check_all="yes"` is set.

---

## Notes

- **Centralized config takes precedence.** If the same directory is
  specified both in a centralized configuration (pushed from the manager)
  and locally in the agent's `ossec.conf`, the centralized one wins and
  overrides the local entry.
- **UNC paths are not supported** for FIM monitoring as of Wazuh 4.13.0+.
- **Config location can also be set on the Wazuh server**, not just per
  agent — useful for pushing the same FIM policy to many agents at once
  via centralized configuration, rather than editing every agent
  individually.
- **Expect some baseline noise** on high-churn directories (e.g. `/tmp`,
  which many system processes write to constantly) — once FIM is enabled
  there, you'll likely see alerts unrelated to your manual test. This is
  realistic behavior, not a bug — tuning which directories/attributes to
  watch (and how tightly) is a normal part of running FIM in production.
