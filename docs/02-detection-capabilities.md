# Detection Capabilities

Deep-dives into the two detection/response capabilities built and
tested in this environment: File Integrity Monitoring (FIM) and a
custom detection rule wired to Active Response.

---

## File Integrity Monitoring (FIM)

FIM watches specified files/directories and alerts on creation,
modification, or deletion. Configured per-agent (or centrally via the
manager), and only takes effect after a restart of the agent service.

Official reference: https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity-monitoring/

### 1. Locate the config file

The `<syscheck>` block lives in `ossec.conf`, in a different path per OS:

| OS | Path |
|---|---|
| Linux | `/var/ossec/etc/ossec.conf` |
| Windows | `C:\Program Files (x86)\ossec-agent\ossec.conf` |
| macOS | `/Library/Ossec/etc/ossec.conf` |

### 2. Edit the `<syscheck>` block

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

**What `check_all="yes"` does** — enables every available attribute
check on files in that directory, not just "did it appear/disappear":
size, permissions, owner (uid)/group (gid), modification time, inode
(catches a file being replaced entirely at the same path), and hash
(MD5/SHA1/SHA256 — catches content changes even when size stays the
same). Without it, you can enable only specific checks (e.g.
`check_size="yes" check_owner="yes"`) — less thorough but less noisy.
`check_all="yes"` is the right choice while testing, for full visibility
into what FIM can actually detect.

**Lowering scan frequency for testing:** `<frequency>` is in seconds,
default `43200` (12 hours) — far too slow for iterative testing. Drop to
`300` (5 minutes) temporarily. **Remember to raise it back** once done
testing — frequent full scans cost real CPU/disk I/O.

### 3. Restart the agent to apply changes

| OS | Command |
|---|---|
| Linux | `sudo systemctl restart wazuh-agent` |
| Windows | `Restart-Service -Name wazuh` (confirm exact service name first) |
| macOS | `/Library/Ossec/bin/wazuh-control restart` |

### 4. Trigger a test event

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

### 5. Check the dashboard

Wait up to your configured `<frequency>` interval (scans are scheduled,
not instant). **Modules → Security events** (or a dedicated Integrity
Monitoring module, depending on version), filtered by the relevant
agent. Expect separate alerts for creation, modification, and deletion —
each carrying the specific changed attribute(s) if `check_all="yes"`.

### FIM notes

- **Centralized config takes precedence** — if the same directory is
  specified both centrally (pushed from the manager) and locally in the
  agent's `ossec.conf`, the centralized one wins.
- **UNC paths are not supported** for FIM monitoring as of Wazuh 4.13.0+.
- **Config can also be set on the Wazuh server**, useful for pushing the
  same FIM policy to many agents at once.
- **Expect baseline noise** on high-churn directories (e.g. `/tmp`,
  which many system processes write to constantly) — this is realistic
  behavior, not a bug; tuning which directories/attributes to watch is a
  normal part of running FIM in production.

---

## Custom detection rule + Active Response blocking

Based on Wazuh's official Proof of Concept guide:
https://documentation.wazuh.com/current/proof-of-concept-guide/blocking-known-malicious-actor.html

**Setup:** Windows 11 + Apache as the "victim" web server, RHEL VM as
the "attacker" endpoint. Manager runs in Docker on AWS EC2.

**Status: detection and rule engine work correctly end-to-end. The
final blocking step does not work**, due to a bug in Wazuh's bundled
`netsh.exe` Active Response binary on Windows — documented below rather
than worked around, since it's a compiled executable that can't be
patched.

### Part 1 — Web server + agent setup (Windows)

Installed Apache (Visual C++ Redistributable → Apache Win64 ZIP →
`C:\Apache24` → installed as a service → allowed through Windows
Firewall). Verified at `http://<WINDOWS_IP>`.

Configured the agent to watch Apache's logs — add to
`C:\Program Files (x86)\ossec-agent\ossec.conf`:
```xml
<localfile>
  <log_format>syslog</log_format>
  <location>C:\Apache24\logs\access_log</location>
</localfile>
```
Restart: `Restart-Service -Name wazuh` (confirm exact service name
first).

### Part 2 — CDB list (AlienVault reputation database)

Run inside the **manager container**, not the raw EC2 host:
```bash
sudo docker exec -it <manager-container-name> bash
```
Download the reputation list:
```bash
curl -o /var/ossec/etc/lists/alienvault_reputation.ipset https://iplists.firehol.org/files/alienvault_reputation.ipset
```
(`wget` isn't installed in this minimal container image — `curl -o`
lowercase, not `-O`, works.)

Download the conversion script and convert:
```bash
curl -o /tmp/iplist-to-cdblist.py https://wazuh.com/resources/iplist-to-cdblist.py
python3 /tmp/iplist-to-cdblist.py /var/ossec/etc/lists/alienvault_reputation.ipset /var/ossec/etc/lists/blacklist-alienvault
rm -f /var/ossec/etc/lists/alienvault_reputation.ipset /tmp/iplist-to-cdblist.py
chown wazuh:wazuh /var/ossec/etc/lists/blacklist-alienvault
```

### Part 3 — Ruleset config (manager)

Add the list to `<ruleset>` in `/var/ossec/etc/ossec.conf`:
```xml
<list>etc/lists/blacklist-alienvault</list>
```
Custom rule in `/var/ossec/etc/rules/local_rules.xml`:
```xml
<group name="attack,">
  <rule id="100100" level="10">
    <if_group>web|attack|attacks</if_group>
    <list field="srcip" lookup="address_match_key">etc/lists/blacklist-alienvault</list>
    <description>IP address found in AlienVault reputation database.</description>
  </rule>
</group>
```

### Part 4 — Active Response block

Add to `/var/ossec/etc/ossec.conf`:
```xml
<command>
  <name>netsh</name>
  <executable>netsh.exe</executable>
  <timeout_allowed>yes</timeout_allowed>
</command>

<active-response>
  <disabled>no</disabled>
  <command>netsh</command>
  <location>local</location>
  <rules_id>100100</rules_id>
  <timeout>60</timeout>
</active-response>
```
Restart the manager (not `systemctl` — this is Docker Compose, no
systemd unit on the host):
```bash
cd ~/wazuh-docker/single-node
sudo docker-compose restart wazuh.manager
```

### Problems encountered, and what was actually wrong

| # | Symptom | Actual cause | Fix |
|---|---|---|---|
| 1 | `wget: command not found` in the manager container | Minimal container image, no wget | Use `curl -o <path> <url>` instead |
| 2 | `curl -O /path url` → "URL rejected: No host part" | Confused curl's `-O` (uppercase, no arg) with `wget -O` (takes a path) | Use lowercase `-o /path <url>` |
| 3 | `nano`/`vi` not found in manager container | Minimal image | `dnf install -y nano` inside the container, or `docker cp` to host and back |
| 4 | Active response never fired at all | The whole `<active-response>` block was left wrapped in an XML comment (`<!-- ... -->`) copied verbatim from a tutorial that showed it commented-out as an example | Remove the surrounding `<!--` and `-->` lines |
| 5 | Rule `100100` never fired from test traffic | This rule is the **AlienVault blacklist** rule — only fires if the source IP is actually on that reputation list. A home LAN/VM IP will never match a real-world threat feed | Realized two different tutorial concepts (blacklist detection vs. "block successive connections") had been merged under one shared rule ID; built a separate custom rule instead (`100102`, below) |
| 6 | Custom frequency rule using `<if_matched_sid>` never fired, even though `firedtimes` was correctly counting up in `wazuh-logtest` | At the time, believed `if_matched_sid` required the referenced rule to itself be frequency-tracking already. **This diagnosis was wrong** — see the corrected version below; the actual fix was switching back to `if_matched_sid` pointed at a proper intermediate rule, not switching away from it | See "Correction" note below — `if_sid` (the fix originally applied here) later caused a hard rule-load failure and had to be reverted |
| 7 | `docker-compose restart wazuh.manager` → "no configuration file provided: not found" | Ran from the wrong directory (`~` instead of `~/wazuh-docker/single-node`) | `cd` into the directory containing `docker-compose.yml` first |
| 8 | Rule fired correctly (dashboard confirmed, level 10, correct description), Active Response *executed* (`netsh.exe` ran, visible in `active-responses.log`), but the connection was never actually blocked | `netsh.exe`'s log showed: `Cannot read 'srcip' from data` — the compiled binary failed to parse the source IP out of the alert JSON, even though `srcip` was clearly present in the payload. Confirmed via deep root-cause investigation (source code review, matched to open upstream GitHub issue) to be a genuine, still-unresolved Wazuh bug, not a config issue — full writeup in `docs/04-known-issues.md` | **Resolved** — replaced with a custom PowerShell Active Response script (`custom-block-ip.ps1`/`.cmd`) that parses `srcip` correctly. See `docs/04-known-issues.md` for the full fix and its own troubleshooting saga (stdin-handling bugs, since fixed) |

### Correction — `if_matched_sid` vs `if_sid` (resolved after further testing)

Issue #6 above was originally "fixed" by switching from `if_matched_sid` to
`if_sid`. **This was incorrect.** Extended testing in a later session
showed:

- `if_sid` + `frequency`/`timeframe` on the same rule causes
  `wazuh-analysisd` to **reject the entire rules file** at load time:
  ```
  ERROR: Invalid use of frequency/context options. Missing if_matched on rule '100102'.
  CRITICAL: Error loading the rules: 'etc/rules/local_rules.xml'.
  ```
  This is unambiguous — Wazuh's own rule engine requires `if_matched_sid`
  (not `if_sid`) whenever `frequency`/`timeframe` context options are
  used. The original "it doesn't fire" symptom that led to switching
  away from `if_matched_sid` was actually caused by something else
  entirely (see below), not by `if_matched_sid` itself being wrong.
- **The real fix**: `if_matched_sid` needs to reference a rule that
  properly participates in frequency correlation. Referencing the raw
  decoder-level base rule (`31108`, level 0) directly doesn't work
  reliably. The working pattern is a **two-rule chain** — a plain
  intermediate rule (no frequency options) promoting the event, then
  the frequency rule referencing that intermediate rule:

```xml
<group name="local,web,">
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
</group>
```

Confirmed working via `wazuh-logtest` (clean two-stage escalation:
`100101` fires on each request, `100102` escalates once the frequency
threshold is met) and via real end-to-end traffic — a 25-request
concurrent burst from the RHEL VM correctly fired the full chain
through to Active Response, visible in the dashboard within the same
second. Full session detail in `docs/03-environment-audit.md`.

**Threshold tuning:** the original `frequency="2" timeframe="60"` (2
requests in 60 seconds) was a very loose testing threshold — easily
triggered by normal browsing. Tuned to `frequency="20" timeframe="3"`
(20+ requests in 3 seconds) for a more realistic scanning/DoS-style
detection, closer to what a production rule would look like.

### The actual working rule: `100101` + `100102`

```xml
<group name="local,web,">
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
</group>
```
This is the rule actually bound to Active Response
(`<rules_id>100102</rules_id>`), not `100100` — see issue #5 above for
why.

### What actually got proven end-to-end

Everything now works, including the final block:

- Apache logs are correctly collected by the agent (`web-accesslog`
  decoder, rule `31108` matching plain requests)
- Intermediate rule `100101` and frequency rule `100102` correctly
  escalate on a real burst of traffic — verified both via
  `wazuh-logtest` (in isolation) and real dashboard alerts from actual
  concurrent curl traffic
- Active Response correctly triggers on `100102` firing, and the
  custom replacement script (`custom-block-ip.cmd`, see
  `docs/04-known-issues.md`) correctly parses `srcip` and applies a
  real Windows Firewall block — confirmed via
  `active-responses.log` showing `Blocked <ip> (rule: Wazuh-Block-<ip>)`
  and independently via a failed `curl` from the source IP during the
  block window
- The block auto-expires after the configured 30-second timeout via
  Wazuh's native stateful Active Response mechanism (the same
  `command":"delete"` callback that was always part of the design), not
  any custom scheduling logic

### Next steps (mostly resolved — remaining items)

1. ~~Check for a newer Wazuh agent version~~ — superseded; root cause
   confirmed to be a genuine upstream bug via source review and a
   matching GitHub issue, not a version-specific quirk (see
   `docs/04-known-issues.md`).
2. ~~Write a custom Active Response script~~ — **done**, see
   `docs/04-known-issues.md` for the full script, its own
   troubleshooting saga, and confirmed end-to-end results.
3. **Test the Linux equivalent (`firewall-drop.sh`)** — still open;
   would confirm whether the Windows-specific `netsh.exe` bug has any
   parallel on the Linux Active Response path, or is isolated to the
   Windows binary.
