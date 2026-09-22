# Wazuh SIEM PoC — Consolidated Notes

## Purpose of this document

This consolidates everything covered across the PoC-building sessions:
how the scope evolved, the core concepts underpinning why the system
works the way it does, and exactly what was built and why. Intended as
a single reference — either for onboarding someone else onto this
work, or for your own future self revisiting it.

---

## 1. Scope — how we got here

### The original ask

The workplace wanted a PoC demonstrating Wazuh integration with a
third-party system, specifically simulating the **receiving SIEM
side** of the CBC (Cell Broadcast) / Emergency Warning System's LLD
requirement — logs need to reach the telco operator's SIEM before the
real integration happens.

### Two demos were initially scoped

1. **Demo 1 — "How a SIEM fundamentally works"**: spoof some alerts,
   show raw log → decode → rule → alert → dashboard, entirely
   self-contained in Wazuh's own built-in stack.
2. **Demo 2 — "Integration + normalization"**: the full Kafka →
   Benthos → Elasticsearch → Kibana pipeline, proving multi-source
   normalization at scale.

### Decision: Demo 2 dropped

**Demo 2 (Kafka/Benthos/Elasticsearch/Kibana) was deprioritized and
removed from scope entirely.** Reasoning (see Section 5 for the full
technical comparison): Wazuh already normalizes incoming data via its
own decoder engine. For a PoC with one source category, one
destination, and no cost-per-GB ingestion pressure, adding a separate
normalization stack duplicates work Wazuh already does natively.
Benthos-style pipelines earn their place with fan-out to multiple
destinations, ingestion-cost-driven filtering, or cross-event
correlation — none of which apply here yet.

**Current state: the entire PoC lives inside Wazuh's own stack.**

### Scope within Demo 1 — narrowed twice

- **First pass**: covered all three of the LLD's named syslog
  categories — login attempts, configuration changes, and errors —
  fully built and verified (see Section 3).
- **Second, current narrowing**: the MNO (mobile network operator,
  i.e. the actual end customer) is **only concerned with authentication
  events** — login success, login failure, SSH attempts, etc.
  Configuration-change and generic-error tracking are out of scope
  going forward.
- **Scope also expanded sideways**: beyond OS-level SSH auth, the PoC
  now needs to cover authentication events from three additional
  software components already present in the real Element Manager
  stack: **PostgreSQL, Oracle DB, and Keycloak** (see Section 4).

---

## 2. Foundational concepts

### A SIEM does not generate logs — it consumes them

This was the single most important conceptual correction of the whole
project. Logging is not a SIEM feature — it's a basic OS/application
capability that exists independent of any security tooling:

- **Linux systems run a syslog daemon (rsyslog, or syslog-ng) by
  default**, as a core OS service, with zero relation to security
  monitoring. It collects messages from the kernel and applications
  and writes them to files under `/var/log/` — this has been happening
  the entire time, invisibly, regardless of whether Wazuh exists.
- **A SIEM's job starts *after* a log already exists.** Wazuh reads
  logs that some other system already produced — it never creates the
  underlying event data itself.

### What a SIEM actually adds, conceptually

Four things, layered on top of already-existing logs:
1. **Reads** — via an installed agent watching local files, or by
   listening for forwarded syslog directly (no agent required)
2. **Decodes** — turns unstructured text into structured fields (e.g.
   `"Failed password for admin from 1.2.3.4"` → `{user: admin, srcip:
   1.2.3.4}`)
3. **Correlates and scores** — evaluates structured events against
   rules, assigning severity (a single failed login is noise; ten in a
   row from one IP, or a failure-then-success pattern, is a scored
   alert)
4. **Stores and surfaces** — pushes results into a searchable,
   visualizable dashboard

### Decoder vs. Rule — a critical distinction

These are two separate stages, and a failure in one looks identical to
a failure in the other unless you check both explicitly:
- **Decoders** do pattern-matching/extraction on raw text. If no
  decoder matches, the event never even reaches rule evaluation
  ("No decoder matched" in `wazuh-logtest`).
- **Rules** evaluate the *already-decoded* fields and decide
  severity/alerting. A rule can be written perfectly correctly and
  still never fire if the decoder step upstream silently failed.

This distinction caused the majority of the debugging time throughout
this project (see Section 3's decoder history for the specifics).

### Source attribution without agent enrollment

Syslog-forwarded events (no agent installed on the sending host) don't
get their own distinct "agent" identity on the dashboard — they all
land under agent `wazuh.manager` / `000`. However, two fields are
**already populated automatically** on every syslog-sourced alert,
with no custom decoder work required:
- `predecoder.hostname` — the sender's hostname, parsed from the
  syslog header
- `location` — the sender's actual source IP

Adding these as visible columns in the dashboard (Threat Hunting →
field picker) is sufficient to distinguish multiple sources, even
though they share one "agent" identity.

### Normalization: two different approaches, compared

| | Wazuh decoders (what we use) | Benthos (dropped from scope) |
|---|---|---|
| Where it happens | Inside Wazuh, at ingestion, via XML pattern-matching | A separate pipeline stage, via Bloblang scripting |
| Normalizes into | Wazuh's own fixed field set (`srcuser`, `srcip`, `status`, etc.) | Any custom schema you design |
| Extra infrastructure | None — built into Wazuh already | Kafka + Benthos + Elasticsearch, a separate stack |
| Worth it when... | Single destination, modest source variety | Fan-out to multiple destinations, cost-per-GB ingestion pricing (e.g. Splunk), or cross-event correlation needs |

**Both approaches solve the same underlying problem** — making
differently-shaped raw data look consistent regardless of source.
Wazuh decoders extract into standard field names (`srcuser`, `srcip`,
`status`, `action` — documented in the header comment of
`local_decoder.xml`) directly at ingestion; Benthos would do the same
reshaping externally, before the SIEM ever sees the data. For this
PoC's shape (one destination, no vendor-portability requirement, no
ingestion cost pressure), the decoder-only approach is the correct,
simpler choice — not a shortcut.

---

## 3. What was built — Demo 1 components

### 3.1 FIM (File Integrity Monitoring) — bonus capability

Demonstrates a different detection category entirely (file-level
changes, not log-based). Configured on `AdamsLaptop`'s Downloads
folder with `check_all="yes"` and `realtime="yes"`.

**Key gotcha:** realtime monitoring requires an initial baseline scan
to complete before it can detect changes — confirmed via
`ossec.log`'s `(6012): Real-time file integrity monitoring started`
line. Testing before this line appears will silently fail to produce
any alert, easily mistaken for a broken config.

### 3.2 SSH brute-force injection (`demo1-ssh-bruteforce-injection.sh`)

Uses `logger` to write crafted lines matching sshd's real log format
locally on the RHEL VM — no actual network attack traffic involved, no
dependency on VM-to-VM network reachability (relevant since the VM has
been running in NAT mode due to bridged-networking issues on the work
network).

**Result, using Wazuh's entirely built-in ruleset (no custom rules
needed):** a full escalation chain — repeated failed logins (rule
`5710`) → Wazuh's own brute-force correlation (`5712`) → a
failure-then-success pattern correlation (`40112`, level 12) → the
attacker's subsequent `sudo` usage (`5402`/`5403`). This remains
**fully in scope** under the auth-only narrowing.

### 3.3 Configuration-change and error injection — now out of scope

`demo1-config-and-error-injection.sh` and the custom
`demo-app-error`/`demo-app-error-detail` decoder + rules
(`100200`/`100201`) were built to cover the LLD's config-change and
error categories. **These are no longer needed** given the
auth-only narrowing, but are documented in full (including the
multi-stage decoder-syntax debugging saga) in
`wazuh-cbc-poc-changes.md`, in case config/error scope returns later.

### 3.4 Manager: remote syslog listener

Required to receive syslog from unmanaged hosts (simulating CBC
nodes) — a separate mechanism from normal agent enrollment (port
1514, encrypted). Configured via a `<remote>` block:
```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>0.0.0.0/0</allowed-ips>
</remote>
```
**Known limitation:** TCP was configured and tested, but the manager
consistently came up listening on UDP regardless of block ordering —
a documented limitation in this Wazuh version. Committed to UDP as the
working, sufficient choice for this PoC.

**Also required:** `<logall>yes</logall>` in `<global>`, otherwise
received-but-unmatched events are fully processed but never appear in
`archives.log` — easy to misread as "not arriving" when it's actually
just "not being logged for visibility."

### 3.5 rsyslog forwarding (direct/live path)

`/etc/rsyslog.d/90-wazuh-forward.conf` on the RHEL VM:
```
*.* @<manager-ip>:514
```
**Under the auth-only narrowing, this should be scoped down** to:
```
auth.*;authpriv.* @<manager-ip>:514
```
so the VM only forwards authentication-relevant facilities, not
everything rsyslog handles.

**Confirmed overlap behavior:** any event sent this way shows up
*twice* on the dashboard — once via this direct forward (attributed to
`wazuh.manager`, with the real sender's hostname/IP visible via
`predecoder.hostname`/`location`), and once via the RHEL VM's own
already-installed Wazuh agent independently picking up the same local
event via journald (attributed to `RHEL-8.10-VM`). This is expected,
not a bug — two legitimate collection mechanisms running
simultaneously.

### 3.6 Multi-source hostname simulation

Standard `logger` can't override the hostname field in an outgoing
syslog packet (rsyslog always fills it with the real OS hostname).
Worked around by constructing raw RFC 3164 syslog packets by hand and
sending via `nc -u` directly:
```bash
line="<${pri}>$(date "+%b %e %H:%M:%S") ${fake_hostname} ${tag}: ${message}"
echo -n "$line" | nc -u -w1 <manager-ip> 514
```
Confirmed working — three simulated hostnames (`cbc-node-01/02/03`)
each independently produced distinguishable alerts, visible via the
`predecoder.hostname` column. This remains relevant for simulating
"multiple VMs" from one real machine.

### 3.7 File-based ingestion (`imfile`)

Added specifically because the workplace supervisor requested
`.txt`/`.log` files as a visual artifact, and wanted the SIEM
integration demonstrated via file-based log delivery (closer to how a
real CBC system would continuously write to its own log file).

`/etc/rsyslog.d/95-cbc-imfile.conf`, bound to a **dedicated ruleset**
(`cbcForward`) so these files aren't double-forwarded by the existing
catch-all rule:
```
module(load="imfile")

ruleset(name="cbcForward") {
    action(type="omfwd" target="<manager-ip>" port="514" protocol="udp")
}

input(type="imfile"
      File="/var/log/cbc-demo/cbc-auth.log"
      Tag="sshd:"
      Facility="auth"
      Severity="notice"
      Ruleset="cbcForward")
```
**Under auth-only scope, only the `cbc-auth.log` input is needed** —
the config-change and error file inputs should be removed.

**Two critical gotchas discovered (both genuinely non-obvious):**

1. **Live/working files must contain ONLY the raw message** — no
   embedded fake timestamp/hostname. `imfile` wraps each line with its
   **own real envelope** before forwarding; embedding a fake header in
   the file content produces a double-wrapped message that no decoder
   can parse. (The separate `*-sample.log` *display* files correctly
   keep full header formatting, since those are purely a visual
   artifact, never fed through `imfile`.)

2. **The `Tag` parameter needs a trailing colon** (`Tag="sshd:"`, not
   `Tag="sshd"`). Wazuh's pre-decoder expects the standard
   `hostname program_name: message` convention — the colon is the
   delimiter between program name and message. Without it, the
   pre-decoder can't correctly split the two, and the event silently
   fails to decode even though it's successfully received.

**Reliable verification method** (worth reusing for anything new):
append a message with a unique, never-before-seen marker string,
`grep` for that exact string in `archives.log` on the manager
immediately — unambiguous, avoids the timezone/duplicate-alert
confusion that made manual dashboard-reading unreliable during live
testing.

---

## 4. New sources in scope — PostgreSQL, Oracle, Keycloak

All three produce genuine authentication log data, but with
meaningfully different formats and delivery mechanisms.

### 4.1 PostgreSQL — straightforward, same pattern as existing work

**Format:** plain text by default (CSV also available via
`log_destination = 'csvlog'`, not used here since it's a different
ingestion approach than what's already built).
```
2026-09-18 10:16:01.456 UTC [4022] admin@cbc_db FATAL:  password authentication failed for user "admin"
```
**Setup required:**
```
log_connections = on
log_disconnections = on
log_line_prefix = '%m [%p] %q%u@%d '
```
**Delivery:** identical to the existing `imfile` pattern — failed
logins are logged by default (no special config needed, it's baseline
`FATAL`-level error logging); successful connections/disconnections
need the two settings above explicitly enabled. Everything writes
directly and synchronously to the log file in real time — no polling,
no intermediate data store.

### 4.2 Keycloak — straightforward, best structural fit

**Format:** hybrid — standard text log prefix, followed by a
`key=value` structured payload:
```
2026-09-18 10:20:15,342 WARN  [org.keycloak.events] type=LOGIN_ERROR, userId=null, ipAddress=203.0.113.55, username=admin
```
This structured tail is genuinely easier to decode reliably than
prose-style messages (like sshd's), since you pattern-match on exact
field names (`type=`, `userId=`, `ipAddress=`) rather than fragile
phrase matching.

**Delivery:** same `imfile` pattern — tail Keycloak's log file
directly. Also relevant: Keycloak is already part of the real Element
Manager BOM ("Identity and access management"), making it the most
production-realistic of the three sources.

### 4.3 Oracle DB — the genuinely different case

**Format depends entirely on audit mode.** Given the auth-only,
current-best-practice scope, **Unified Auditing** (Oracle 12c+) is the
relevant mode — but unlike the other two sources, **unified audit
trail data lives in a database view (`UNIFIED_AUDIT_TRAIL`), not a
flat file.** There's nothing to `imfile`-tail directly; an export step
is required first.

**The key architectural problem: successful and failed logins need
different mechanisms, because Oracle has no way to react automatically
the instant a row lands in the audit trail** (it's not a regular table
— you can't attach a conventional `AFTER INSERT` trigger to it).

| Event | Mechanism | Real-time? |
|---|---|---|
| Successful login | `AFTER LOGON ON DATABASE` trigger | ✅ Immediate, event-driven — fires as part of native session establishment |
| Successful logout | `BEFORE LOGOFF ON DATABASE` trigger | ✅ Immediate, event-driven |
| **Failed login** | Query `UNIFIED_AUDIT_TRAIL` | ❌ **No trigger equivalent exists** — a failed authentication never establishes a session, so `AFTER LOGON` structurally cannot fire for it. Must be captured via a scheduled poll of the audit trail. |

**Practical design (planned, not yet built):**
1. Logon/logoff triggers write directly to a flat file (via
   `UTL_FILE`) the instant they fire — no delay
2. A lightweight scheduled job (`DBMS_SCHEDULER`, e.g. every 10-30s)
   queries `UNIFIED_AUDIT_TRAIL` for new failed-login rows since the
   last check (tracked by `EVENT_TIMESTAMP` or
   `AUDIT_SEQUENCE_NUM`), appending them to the same output file
3. Both paths converge on one flat file, which then feeds into the
   existing `imfile` pattern exactly like the other two sources

**Known risk worth confirming before building:** a documented Oracle
12.1 bug caused `UNIFIED_AUDIT_TRAIL` to silently fail to capture
logon failures at all (fixed in 12.2 via patch 19383839) — worth
explicitly confirming the Oracle version and testing capture before
relying on this.

---

## 5. Current end-to-end picture (auth-only scope)

```
                          ┌─ PostgreSQL log file ─────┐
                          ├─ Keycloak log file ───────┤
                          ├─ Oracle export file        │
                          │  (triggers + polling job) ─┤
                          └─ RHEL VM auth.log/sshd ────┘
                                       │
                              rsyslog (imfile, per source)
                                       │
                          UDP 514, syslog protocol
                                       │
                              Wazuh MANAGER
                          (remote syslog listener)
                                       │
                     Decode (built-in sshd/sudo decoders,
                      or custom decoders per new source)
                                       │
                        Rule evaluation + severity scoring
                                       │
                          Wazuh Dashboard (Threat Hunting)
                     — source distinguishable via predecoder.hostname
```

---

## 6. Open items / not yet built

- [ ] Trim existing scripts/configs to auth-only (remove config-change
      and error injection, narrow rsyslog facility forwarding to
      `auth.*;authpriv.*`, remove non-auth `imfile` inputs)
- [ ] Build PostgreSQL log ingestion (enable logging, wire into
      `imfile`, build decoder extracting `srcuser`/status)
- [ ] Build Keycloak log ingestion (same pattern, decoder for the
      `key=value` tail)
- [ ] Build Oracle logon/logoff triggers + `UTL_FILE` export
- [ ] Build Oracle failed-login polling job (`DBMS_SCHEDULER`) +
      confirm unified auditing is genuinely capturing failures (check
      Oracle version against the known 12.1 bug)
- [ ] Confirm with CBC vendor: RFC 3164 vs RFC 5424 syslog format
      (current work assumes RFC 3164 throughout)
- [ ] Production hardening notes (for the eventual real deployment,
      not this PoC): TLS-wrapped syslog instead of plain UDP 514;
      scope `<allowed-ips>` to actual node IPs instead of `0.0.0.0/0`
- [ ] `xton_r` (read-only) vs `xton_rw` (read-write) permission-denial
      scenario — not yet demoed, would strengthen the auth-event story
      by showing an authorization failure distinct from an
      authentication failure
