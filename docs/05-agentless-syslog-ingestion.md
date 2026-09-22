# Agentless Syslog Ingestion — Custom Decoder + Remote Listener

A demo showing Wazuh can receive and correctly parse logs from
**unmanaged remote hosts** — machines with no Wazuh agent installed at
all — via plain syslog forwarding, rather than requiring every log
source to run an agent. This is a genuinely different ingestion path
from everything else in this repo, which has all been agent-based
(FIM, Active Response, log collection via `<localfile>`).

Covers: a custom decoder/rule for generic application errors, a remote
syslog listener on the manager, rsyslog forwarding configuration on the
RHEL VM (simulating an external unmanaged system), multi-source
simulation, and file-based (`imfile`) log ingestion.

Scoped to demonstrating three general syslog categories: **login
attempts**, **configuration changes**, and **application/system
errors**.

---

## 1. Custom decoder + rule: generic application/system errors

### Why this was needed

Wazuh's default ruleset has built-in decoders for well-known formats
(sshd, sudo, etc.), but no decoder recognizes arbitrary application
error text like `databaseConnect: connection timed out`. Without a
decoder, `wazuh-analysisd` reports **"No decoder matched"** and the
event never reaches rule evaluation — confirmed via `wazuh-logtest`.

### Final working configuration

**`/var/ossec/etc/decoders/local_decoder.xml`:**
```xml
<decoder name="demo-app-error">
  <program_name>vmuser</program_name>
</decoder>

<decoder name="demo-app-error-detail">
  <parent>demo-app-error</parent>
  <use_own_name>true</use_own_name>
  <prematch type="pcre2">databaseConnect:|diskMonitor:</prematch>
  <regex offset="after_prematch" type="pcre2">\s*(.+)$</regex>
  <order>description</order>
</decoder>
```

**`/var/ossec/etc/rules/local_rules.xml`:**
```xml
<group name="local,errors,">
  <rule id="100200" level="7">
    <decoded_as>demo-app-error-detail</decoded_as>
    <description>Demo: application/system error detected</description>
  </rule>

  <rule id="100201" level="13">
    <if_matched_sid>100200</if_matched_sid>
    <match>diskMonitor</match>
    <description>Demo: critical system resource error (disk pressure)</description>
  </rule>
</group>
```

**Note the `if_matched_sid` here, not `if_sid`** — this matches the
hard-learned lesson documented in `docs/02-detection-capabilities.md`
and `docs/03-environment-audit.md`: `if_sid` combined with
frequency/context options causes `wazuh-analysisd` to reject the entire
rules file outright, not just this one rule.

### Debugging history (kept for future reference — several wrong turns)

1. **First attempt** — single decoder with
   `<prematch>^databaseConnect:|^diskMonitor:</prematch>`. Worked in
   `wazuh-logtest` when given the message with **no syslog header**,
   but failed ("No decoder matched") against the real, full
   syslog-formatted line. Root cause: `prematch` is documented to only
   test the log **after** the syslog-like header is stripped — testing
   without a header meant there was nothing to strip, masking the real
   issue.

2. **Removed the `^` anchors**, assuming `prematch` was anchored to
   position 0 of the full raw string. Still failed — `prematch` doesn't
   behave like a general-purpose "contains" search; anchoring behavior
   is implicit either way for this field.

3. **Attempted `offset="after_parent"`** on a `<regex>` tag — **invalid
   syntax**, caused `wazuh-analysisd` to fail to start entirely
   (`ERROR: (2120): Invalid offset value: 'after_parent'`,
   `CRITICAL: (1202): Configuration error`). Manager was down until
   fixed. **Lesson: always verify the manager restarted cleanly
   (`docker ps`, tail `ossec.log`) after any decoder/rule edit, before
   assuming the change took effect.**

4. **Attempted alternation (`|`) inside `<regex>`** —
   `ERROR: (1452): Syntax error on regex`. Wazuh's decoder `<regex>`
   field is **not** standard PCRE/POSIX by default; alternation and some
   other constructs aren't supported unless `type="pcre2"` is
   explicitly set.

5. **Final fix** — split into parent (`<program_name>`) + child
   (`<parent>` + `<prematch>` + `<regex offset="after_prematch">`), both
   explicitly typed `pcre2`, with `<use_own_name>true</use_own_name>` on
   the child so alerts report the child decoder's name rather than
   silently falling back to the parent's name.

6. **Ownership resets on every manual edit** — `nano` rewrites the file
   as root on save, silently undoing any earlier `chown wazuh:wazuh`.
   Re-applied after every edit:
   ```bash
   chown wazuh:wazuh /var/ossec/etc/decoders/local_decoder.xml /var/ossec/etc/rules/local_rules.xml
   chmod 640 /var/ossec/etc/decoders/local_decoder.xml /var/ossec/etc/rules/local_rules.xml
   ```

---

## 2. Manager: remote syslog listener

### Why this was needed

The manager only listens for its own enrolled agents (port 1514,
encrypted agent protocol) by default. Receiving plain syslog from
unmanaged hosts requires a separate listener.

### Configuration — `/var/ossec/etc/ossec.conf`

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>0.0.0.0/0</allowed-ips>
</remote>
```
Placed **after** the existing secure/agent `<remote>` block.

**Also required, in `<global>`, to make received-but-unmatched events
visible for debugging** (off by default):
```xml
<global>
  <jsonout_output>yes</jsonout_output>
  <alerts_log>yes</alerts_log>
  <logall>yes</logall>
  <logall_json>yes</logall_json>
</global>
```
Without `logall`, events are fully received/processed by `remoted` but
never appear in `archives.log` — easy to misdiagnose as "not arriving"
when it's actually just "not being logged."

### Known limitation — TCP requested, UDP delivered

`<protocol>tcp</protocol>` was configured and tested (block reordering
tried both ways — secure-block-first and syslog-block-first), but
`wazuh-remoted` came up listening on **UDP 514** regardless every time
(`netstat`/`ss` confirmed). This matches a documented/reported
limitation in this Wazuh version where syslog connections default to
UDP under certain block orderings. **Decision: committed to UDP** —
functional and sufficient for this demo; not worth further time chasing
TCP.

### AWS security group requirement

| Port | Protocol | Source |
|---|---|---|
| 514 | **UDP** | The syslog-forwarding host's actual outbound IP |

**Note:** if the forwarding host is on NAT rather than a fixed/bridged
address, the manager sees that host's public/NAT-exit IP, not an
internal address — relevant if `<allowed-ips>` is ever scoped down from
`0.0.0.0/0`.

### Verification commands used throughout

```bash
# Confirm listener is up
sudo docker exec -it single-node-wazuh.manager-1 bash -c "netstat -tulnp | grep 514"

# Confirm manager received a specific event (raw, pre-rule-match)
sudo docker exec -it single-node-wazuh.manager-1 grep "<search-term>" /var/ossec/logs/archives/archives.log

# Confirm alerting
sudo docker exec -it single-node-wazuh.manager-1 grep -A5 "<rule-id>" /var/ossec/logs/alerts/alerts.log

# Confirm packet leaves the source host at all (before blaming the manager/AWS)
sudo tcpdump -i any -n udp port 514
```

---

## 3. Forwarding host: rsyslog forwarding (direct/live events)

### Configuration — `/etc/rsyslog.d/90-wazuh-forward.conf`

```
*.* @<manager-elastic-ip>:514
```
Single `@` = UDP (matches the manager's actual listener). Forwards
everything rsyslog handles — broad for demo purposes; would be narrowed
in production (e.g. `auth.*;authpriv.*`).

### Behavior confirmed

- Every event sent this way is picked up **twice** on the dashboard:
  once via this direct syslog-forward path (shows up under the
  manager's own agent entry, with the real sender hostname/IP visible
  in `predecoder.hostname` / `location`), and once via the host's own
  **already-installed Wazuh agent** independently collecting the same
  local event via journald (shows up under that host's own agent
  entry). This is expected overlap, not a bug — both mechanisms
  legitimately run simultaneously when the forwarding host also happens
  to be an enrolled agent.

---

## 4. Source/hostname attribution (multi-node simulation)

### Key finding

Syslog-forwarded events are **not** attributed to a distinct per-source
"agent" — they all land under the manager's own agent entry, since
there's no agent enrollment involved. However, the original sender's
hostname and IP **are** captured, in two fields already present on
every syslog-sourced alert:
- `predecoder.hostname` — the hostname from the syslog message header
- `location` — the sender's actual source IP

**No custom decoder work was needed** for this — both fields are
populated automatically. Fix was purely a dashboard config step: add
`predecoder.hostname` as a visible column in **Threat Hunting** (field
picker → search field name → "+" icon).

### Simulating multiple distinct remote sources from one host

Standard `logger` can't override the hostname field (rsyslog always
fills it with the real OS hostname when forwarding). Worked around by
constructing raw RFC 3164 syslog packets by hand and sending directly
via `nc -u`, bypassing rsyslog's own forwarding for this specific
purpose:

```bash
line="<${pri}>$(date "+%b %e %H:%M:%S") ${fake_hostname} ${tag}: ${message}"
echo -n "$line" | nc -u -w1 <manager-ip> 514
```
Confirmed working — three distinct simulated hostnames each
independently produced their own login/config-change/error triplet,
all correctly decoded, all visually distinguishable via
`predecoder.hostname` in the dashboard.

---

## 5. File-based ingestion (`imfile`) — simulating a real remote log file

### Why this was added

To demonstrate file-based logs being sent into the SIEM — closer to how
a real remote system would behave (writing continuously to its own log
file, with forwarding as a secondary/automatic action), versus the
direct `logger`/`nc` injection used earlier.

### Configuration — `/etc/rsyslog.d/95-imfile-demo.conf`

```
module(load="imfile")

ruleset(name="demoForward") {
    action(type="omfwd" target="<manager-elastic-ip>" port="514" protocol="udp")
}

input(type="imfile"
      File="/var/log/demo-remote/demo-auth.log"
      Tag="sshd:"
      Facility="auth"
      Severity="notice"
      Ruleset="demoForward")

input(type="imfile"
      File="/var/log/demo-remote/demo-config.log"
      Tag="sudo:"
      Facility="authpriv"
      Severity="notice"
      Ruleset="demoForward")

input(type="imfile"
      File="/var/log/demo-remote/demo-errors.log"
      Tag="vmuser:"
      Facility="daemon"
      Severity="error"
      Ruleset="demoForward")
```

Bound to a **dedicated ruleset** (`demoForward`) so these three files
are not double-forwarded by the existing `90-wazuh-forward.conf`
catch-all — inputs bound to a named ruleset skip the default ruleset
entirely.

### Critical gotcha #1 — file content must NOT include a fake syslog header

Initial sample/working files were written to already look like
complete syslog lines (`Sep 17 23:02:58 hostname sshd[2274]: Failed
password...`). This is wrong for `imfile`: rsyslog wraps each raw file
line with **its own real envelope** (actual hostname, actual timestamp,
the `Tag`/`Facility`/`Severity` from the config) before forwarding.
Embedding a fake header in the file content produces a **double-wrapped**
message — confirmed directly in `archives.log`:
```
hostname->1.2.3.4 Sep 17 23:02:58 hostname sshd Sep 17 23:02:58 hostname sshd[2274]: Failed password...
```
The manager's predecoder correctly strips its own real outer envelope,
but the leftover text still starts with the embedded fake
timestamp/hostname, which no decoder recognizes.

**Fix:** live/working files fed to `imfile` must contain **only the raw
message** — no embedded timestamp, hostname, or tag:
```
Failed password for invalid user admin from 203.0.113.55 port 51201 ssh2
```
(A separate `*-sample.log` file, meant purely as a visual artifact of
"what this log looks like on disk," correctly keeps the full header
format — that distinction matters and was easy to conflate.)

### Critical gotcha #2 — the `Tag` parameter needs a trailing colon

Even after fixing gotcha #1, events still failed to reach
`archives.log` in a parseable form. Root cause: Wazuh's pre-decoder
expects the standard syslog convention `hostname program_name: message`
— the **colon immediately after the program name** is what delimits
program name from message. `Tag="sshd"` (no colon) produces
`hostname sshd message` (bare space, no delimiter), which the
pre-decoder can't correctly split.

**Fix:** always include the trailing colon in `Tag`:
```
Tag="sshd:"
Tag="sudo:"
Tag="vmuser:"
```

### Verification approach used (recommended for future changes)

Given how easy it was to misread overlapping/duplicate-looking alerts
across timezones during live testing, the most reliable verification
method was:
1. Append a message with a **unique, never-before-seen marker string**
   (e.g. `VERIFYTEST-12345`)
2. Immediately `grep` for that exact string in `archives.log` on the
   manager — unambiguous yes/no, no timestamp math required
3. Only then check the dashboard, searching for the same marker string
   directly rather than trying to eyeball a time window

---

## Current file inventory (forwarding host)

| Path | Purpose |
|---|---|
| `/etc/rsyslog.d/90-wazuh-forward.conf` | Direct/live forwarding (`*.* @manager:514`) |
| `/etc/rsyslog.d/95-imfile-demo.conf` | `imfile` watchers for the three demo log files |
| `/var/log/demo-remote/demo-auth.log` | Live-tailed, message-only content |
| `/var/log/demo-remote/demo-config.log` | Live-tailed, message-only content |
| `/var/log/demo-remote/demo-errors.log` | Live-tailed, message-only content |

## Current file inventory (Wazuh manager)

| Path | Purpose |
|---|---|
| `/var/ossec/etc/decoders/local_decoder.xml` | `demo-app-error` / `demo-app-error-detail` decoders |
| `/var/ossec/etc/rules/local_rules.xml` | Rules `100200`/`100201`, plus pre-existing `100100`/`100101`/`100102` |
| `/var/ossec/etc/ossec.conf` | `<remote>` syslog block (UDP 514), `<global>` with `logall` enabled |

## Rule IDs referenced/produced during this demo

| Rule ID | Level | Description | Built-in / Custom |
|---|---|---|---|
| 5710 | 5 | sshd: attempt to login using a non-existent user | Built-in |
| 5712 | 10 | sshd: brute force trying to get access, non-existent user | Built-in |
| 5760 | 5 | sshd: authentication failed (generic) | Built-in |
| 40112 | 12 | Multiple authentication failures followed by a success | Built-in (correlation) |
| 5402 | 3 | Successful sudo to ROOT executed | Built-in |
| 5403 | 4 | First time user executed sudo | Built-in |
| 100200 | 7 | Demo: application/system error detected | Custom |
| 100201 | 13 | Demo: critical system resource error (disk pressure) | Custom |

---

## Open items / not yet done

- Confirm RFC 3164 vs RFC 5424 syslog format assumptions against a real
  target system, if this is ever extended beyond a demo
- Production hardening: TLS-wrapped syslog (RFC 5425, typically port
  6514) instead of plain UDP 514; scope `<allowed-ips>` to actual known
  source IPs instead of `0.0.0.0/0`
- Read-only vs. read-write permission-denial scenarios not yet demoed —
  only successful write-type actions covered so far
- "Dead man's switch" / absence-of-activity detection not covered —
  everything above is presence-of-bad-event detection only
