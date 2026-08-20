# Blocking a known malicious actor — setup notes

Based on Wazuh's official Proof of Concept guide:
https://documentation.wazuh.com/current/proof-of-concept-guide/blocking-known-malicious-actor.html

This attempt used a Windows 11 + Apache endpoint as the "victim" web
server, and a RHEL VM as the "attacker" endpoint. The manager runs in
Docker on AWS EC2.

**Status: detection and rule engine work correctly end-to-end. The final
blocking step does not work, due to a bug in Wazuh's bundled
`netsh.exe` Active Response binary on Windows — documented below rather
than worked around, since it's a compiled executable we can't patch.**

---

## Part 1 — Web server + agent setup (Windows)

### Install Apache

Followed the official guide: Visual C++ Redistributable → Apache Win64
ZIP → extracted to `C:\Apache24` → installed as a service → allowed
through Windows Firewall.

Verify:
```
http://<WINDOWS_IP>
```
Should show the default "It works!" page.

### Configure the Wazuh agent to watch Apache's logs

Edit (as Administrator):
```
C:\Program Files (x86)\ossec-agent\ossec.conf
```

Add:
```xml
<localfile>
  <log_format>syslog</log_format>
  <location>C:\Apache24\logs\access_log</location>
</localfile>
```

Restart the agent (PowerShell, as Administrator):
```powershell
Restart-Service -Name wazuh
```
(confirm the actual service name first: `Get-Service | Where-Object {$_.Name -like "*wazuh*"}`)

## Part 2 — CDB list (AlienVault reputation database)

All of the following steps run **inside the manager container**, not on
the raw EC2 host — the container's filesystem is separate from the
host's.

```bash
sudo docker exec -it <manager-container-name> bash
```

Download the reputation list:
```bash
curl -o /var/ossec/etc/lists/alienvault_reputation.ipset https://iplists.firehol.org/files/alienvault_reputation.ipset
```
(`wget` isn't installed in this container image — `curl` works, but note
curl's `-O` [uppercase] and `-o` [lowercase] behave differently: uppercase
saves under the remote filename and takes no argument, lowercase saves to
the exact path given. Using uppercase `-O` with a path argument produces
a confusing "URL rejected: No host part" error.)

Download the ipset→cdb conversion script:
```bash
curl -o /tmp/iplist-to-cdblist.py https://wazuh.com/resources/iplist-to-cdblist.py
```

Convert:
```bash
python3 /tmp/iplist-to-cdblist.py /var/ossec/etc/lists/alienvault_reputation.ipset /var/ossec/etc/lists/blacklist-alienvault
```

Clean up the intermediate files and fix ownership:
```bash
rm -f /var/ossec/etc/lists/alienvault_reputation.ipset /tmp/iplist-to-cdblist.py
chown wazuh:wazuh /var/ossec/etc/lists/blacklist-alienvault
```

## Part 3 — Ruleset config (manager)

Add the list to `<ruleset>` in `/var/ossec/etc/ossec.conf`:
```xml
<list>etc/lists/blacklist-alienvault</list>
```

Add a custom rule in `/var/ossec/etc/rules/local_rules.xml`:
```xml
<group name="attack,">
  <rule id="100100" level="10">
    <if_group>web|attack|attacks</if_group>
    <list field="srcip" lookup="address_match_key">etc/lists/blacklist-alienvault</list>
    <description>IP address found in AlienVault reputation database.</description>
  </rule>
</group>
```

## Part 4 — Active Response block

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

Restart the manager:
```bash
cd ~/wazuh-docker/single-node
sudo docker-compose restart wazuh.manager
```

**Not `systemctl restart wazuh-manager`** — this is a Docker Compose
deployment, not a bare-metal install, so there's no systemd unit for
Wazuh on the host. `docker-compose` also needs to be run from the
directory containing `docker-compose.yml`, or it can't resolve the
service name.

---

## Problems encountered, and what was actually wrong

| # | Symptom | Actual cause | Fix |
|---|---|---|---|
| 1 | `wget: command not found` in the manager container | Minimal container image, no wget | Use `curl -o <path> <url>` instead |
| 2 | `curl -O /path url` → "URL rejected: No host part" | Confused curl's `-O` (uppercase, no arg) with `wget -O` (takes a path) | Use lowercase `-o /path <url>` |
| 3 | `nano`/`vi` not found in manager container | Minimal image | `dnf install -y nano` inside the container, or edit via `docker cp` to host and back |
| 4 | Active response never fired at all | The whole `<active-response>` block was left wrapped in an XML comment (`<!-- ... -->`) copied verbatim from a tutorial that showed it commented-out as an example | Remove the surrounding `<!--` and `-->` lines |
| 5 | Rule `100100` never fired from test traffic | This rule is the **AlienVault blacklist** rule — it only fires if the source IP is actually on that reputation list. A home LAN/VM IP will never match a real-world threat feed | Realized two different tutorial concepts (blacklist detection vs. "block successive connections") had been merged under one shared rule ID; built a separate custom rule instead (see below) |
| 6 | Custom frequency rule using `<if_matched_sid>` never fired, even though `firedtimes` was correctly counting up in `wazuh-logtest` | `if_matched_sid` requires the *referenced* rule to itself be a frequency-tracking rule already — a level-0 base rule like the Apache "ignored URL" rule doesn't participate in this the way expected | Switched to `<if_sid>` + `<same_source_ip />` on the new rule instead — checks each event against the base rule independently, tracks frequency on the new rule itself |
| 7 | `docker-compose restart wazuh.manager` → "no configuration file provided: not found" | Ran from the wrong directory (`~` instead of `~/wazuh-docker/single-node`) | `cd` into the directory containing `docker-compose.yml` first |
| 8 | Rule fired correctly (confirmed in dashboard, level 10, correct description), Active Response *executed* (`netsh.exe` ran, visible in `active-responses.log`), but the connection was never actually blocked | `netsh.exe`'s log showed: `Cannot read 'srcip' from data` — the compiled binary failed to parse the source IP out of the alert JSON it was given, even though `srcip` was clearly present in the payload. This is a bug/version-mismatch inside Wazuh's own pre-compiled Windows Active Response binary, not a config issue | Not resolved. `netsh.exe` is a compiled `.exe` (confirmed via hex/byte inspection — starts with the `MZ` PE header), not an editable script, so the internal JSON-parsing bug can't be patched directly. Documented as a known limitation. |

## What actually got proven end-to-end

Despite issue #8, everything up to the final block genuinely works:

- Apache logs are correctly collected by the agent (`web-accesslog`
  decoder, rule `31108` matching plain requests)
- A custom frequency-based detection rule (`100102`, `frequency="2"
  timeframe="60"`) correctly identifies repeated requests from the same
  source within a time window — verified both via `wazuh-logtest` (in
  isolation) and real dashboard alerts from actual curl traffic
- Active Response correctly triggers on that rule firing, and the
  configured command (`netsh.exe`) does execute on the Windows agent
- The only broken link is inside Wazuh's shipped Windows binary's own
  JSON parsing — confirmed via its own log output
  (`active-response/active-responses.log`)

## Possible next steps (not pursued so far)

1. **Check for a newer Wazuh agent version** — this may be a
   version-specific bug already fixed upstream. Compare
   `C:\Program Files (x86)\ossec-agent\VERSION` against Wazuh's GitHub
   issues for `netsh.exe` + `srcip`.
2. **Write a custom Active Response script** (PowerShell or batch)
   instead of relying on the bundled binary — parse the alert JSON
   correctly and issue the `netsh advfirewall firewall add rule` command
   directly. More work, but fully within your control.
3. **Test on the Linux side instead** — the RHEL VM's `firewall-drop.sh`
   (the Linux equivalent script) was never actually tested this session;
   it's plausible the Linux path doesn't have the same parsing bug.
