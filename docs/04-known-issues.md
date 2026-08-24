# Known Issues (Upstream — Interim Fixes Applied)

Bugs that have been root-caused in this environment but whose real fix
lives **upstream**, outside this environment's control — tracked here
separately from `03-environment-audit.md`, which covers environment
state and issues fully resolved within this environment itself. An
"interim fix" entry below means the upstream bug is still genuinely
broken in Wazuh itself, but a working replacement has been built and
confirmed here.

---

## Windows Active Response (`netsh.exe`) fails to parse `srcip`

**Status: confirmed upstream bug, unresolved as of Wazuh 4.14.7 —
interim fix (custom script) built, deployed, and confirmed working
end-to-end.**

**Symptom:** the custom detection rule `100102` (multiple web requests
from the same source within 60 seconds) fires correctly and triggers
Active Response as configured, but the actual firewall block never
applies. Every trigger produces:
```
active-response/bin/netsh.exe: Cannot read 'srcip' from data
```
despite `srcip` being genuinely present and correctly populated in the
alert JSON passed to the script.

Full original discovery and setup context:
`docs/02-detection-capabilities.md`.

### Confirmed: bug persists after full environment rework

Re-tested after a significant networking/agent rework (bridged VM
networking, agent re-enrollment, config fixes — see
`docs/03-environment-audit.md`, Issue 4). Result: **identical failure**,
every single trigger:
```
active-response/bin/netsh.exe: Cannot read 'srcip' from data
```
Confirmed via `active-responses.log` across four separate triggers
(`firedtimes: 2` through `6`), all producing the same error.
Cross-checked that no firewall rule was ever actually created:
```powershell
netsh advfirewall firewall show rule name=all | Select-String -Context 2 "wazuh"
```
returned nothing.

**Root cause, precisely stated:** the alert JSON the script receives
genuinely does contain `srcip` correctly —
`"data":{"protocol":"GET","srcip":"192.168.0.201",...}` is present and
correctly populated in the payload. This rules out any possibility that
the alert itself is missing the field; the bug is squarely in how the
compiled `netsh.exe` binary parses/extracts `srcip` from that JSON
structure internally, not a config or data issue on this end.

### Root-cause investigation and upstream engagement

Before building a custom replacement, investigated whether this could
be fixed at the actual source rather than worked around:

- **Ruled out `<expect>srcip</expect>`** as a fix — Wazuh's own FAQ
  confirms this field should be *removed*, not added, for agents 4.2.0+
  using `.exe`-based AR scripts (the modern format, which this setup
  already uses correctly).
- **Confirmed the JSON structure is correct** — `parameters.alert.data.srcip`
  matches Wazuh's own documented working examples for the equivalent
  Linux `firewall-drop` script.
- **Located the actual C source** (`wazuh/src/active-response/netsh.c`)
  — confirmed the failure originates in a call to
  `get_srcip_from_json()`, a shared helper function, not something
  specific to this deployment.
- **Cross-checked official current documentation**
  (documentation.wazuh.com, Active Response configuration guide, v4.14)
  — confirmed the `<command>`/`<active-response>` config already
  follows the documented, correct pattern exactly; nothing missing.
- **Full manager + agent `ossec.conf` review** — read through both
  configs in full (not just grepping for known keywords) specifically
  looking for anything that could explain the failure: duplicate or
  conflicting `<command>`/`<active-response>` blocks, incorrect
  `rules_id`, stray whitelist entries, encoding issues. Found nothing —
  both configs match the documented-correct pattern exactly. (Two
  separate `<global>` blocks exist in the manager config — one for
  general logging settings, one for the Active Response whitelist —
  which is untidy but not a functional error; Wazuh's parser merges
  same-tag blocks correctly, as already relied on elsewhere in this
  config for multiple `<localfile>`/`<command>` entries.)
- **Ruled out the Active Response whitelist as a factor** — the
  manager's `<white_list>` entries (`127.0.0.1`,
  `^localhost.localdomain$`) don't match the actual test traffic source
  (`192.168.0.201`, the VM's real bridged IP), so this isn't
  suppressing anything in this reproduction.
- **Found an identical, still-open GitHub issue**
  ([wazuh/wazuh#21812](https://github.com/wazuh/wazuh/issues/21812),
  filed Feb 2024) — same decoder (`web-accesslog`), same JSON
  structure, same exact error text (`Cannot read 'srcip' from data`),
  same `netsh.exe - add` Active Response trigger pattern. No changelog
  entry found indicating this was resolved between that report and the
  current version (4.14.7).
- **Filed a reproduction comment** on issue #21812 with this
  environment's specific evidence (agent/manager version, config,
  reproduction steps, log excerpts), to contribute independent
  confirmation to the existing report.

### Follow-up: ruled out "firewall disabled" as an alternative explanation

A separate comment on the same issue thread suggested the AR failure in
their case was caused by Windows Firewall being disabled. Checked
whether that applies here before accepting it as a possible explanation
for this reproduction too:
```powershell
Get-Service -Name mpssvc                    # Running
Get-NetFirewallProfile | Select-Object Name, Enabled
# Domain: True, Private: True, Public: True
```
Additionally, an unrelated `netsh advfirewall firewall add rule`
(adding an inbound ICMPv4 allow rule, done earlier while troubleshooting
VM↔host ping) executed successfully and visibly changed ping behavior —
independent proof the firewall was actively enforcing rules throughout
this environment's testing, not just nominally "on."

**Conclusion:** the firewall-disabled explanation does not apply to
this reproduction. This is either a distinct root cause from what that
commenter hit, or the "firewall must be enabled" fix is incomplete/
situational rather than universal. Posted as a follow-up comment on
#21812 to sharpen the bug report.

### Overall conclusion

This is a genuine, confirmed, still-unresolved upstream bug — not a
misconfiguration, version mismatch, environment-specific issue, or
disabled firewall. A custom Active Response script is the correct
interim mitigation while the upstream bug remains open, not a
workaround for a self-inflicted problem.

**Scalability note (raised before building the replacement):** a custom
script solves this for a single lab agent, but does not solve the
underlying production distribution problem — Wazuh's centralized
configuration pushes `ossec.conf` settings to many agents at once, but
does **not** distribute the actual script/binary files themselves,
which still have to be placed individually on every endpoint (via GPO,
SCCM, configuration management tooling, or baked into a base image).
This is an intentional trade-off being made for a single-agent lab
demo, documented here rather than presented as a fully production-ready
fix.

**Next step:** write a custom PowerShell Active Response script
(`custom-block-ip.ps1` + a `.cmd` wrapper, since Wazuh's Windows agent
requires an executable, not a raw `.ps1`, registered in `ossec.conf`)
that parses `srcip` correctly and issues the real
`netsh advfirewall firewall add rule` command, with automatic
expiry/removal on the existing 60s Active Response timeout.

---

## Status: Interim fix built, tested, and confirmed working

A custom Active Response script was built to replace the broken
`netsh.exe` binary. It took three iterations to get right — each
failure taught something specific about how Wazuh's Windows agent
actually invokes Active Response scripts, worth documenting since none
of it was obvious upfront.

### Design

- **`custom-block-ip.cmd`** — thin wrapper registered as the Active
  Response executable. Captures stdin to a temp file via `findstr "^"`,
  passes the file path to the PowerShell script.
- **`custom-block-ip.ps1`** — does the real work: reads the JSON from
  the file, extracts `srcip` from `parameters.alert.data.srcip`,
  validates it looks like a real IPv4 address, and calls the real
  `netsh advfirewall firewall` binary directly (not the
  `New-NetFirewallRule`/`Remove-NetFirewallRule` PowerShell cmdlets —
  see below for why) to add or remove a block rule.
- **Config change**: `ossec.conf`'s `<command>`/`<active-response>`
  blocks updated to point at `custom-block-ip.cmd` instead of
  `netsh.exe`, with `<timeout>` lowered from 60s to **30s** — the block
  auto-expires via Wazuh's own native stateful Active Response
  mechanism (the same script gets called again with
  `"command":"delete"` after the timeout), no custom scheduling logic
  needed.

### Iteration 1 — stdin read via `[Console]::In.ReadToEnd()` hung indefinitely

First version piped JSON directly into PowerShell and read it with
`[Console]::In.ReadToEnd()`. Manual testing (piping JSON in from an
interactive PowerShell/cmd session) **crashed outright** with an
unusual exit code (`0xC06D007E`) before writing anything to the log —
not a clean reproduction of the real invocation context. When actually
triggered by the real Wazuh agent, the script logged `Starting` and
then **hung indefinitely** — no crash, no error, no further log output,
and a stray `powershell.exe` process left running.

### Iteration 2 — switched to `ReadLine()`, still hung

Reasoned that `ReadToEnd()` waits for the input stream to fully close
(EOF), while the JSON is actually delivered as a single
newline-terminated line — so `ReadLine()` should only need to wait for
the newline, not stream closure. Deployed, retested: **identical
hang**, same symptom.

**Isolating the real cause:** manually invoking the script via file
redirection (`cmd /c "custom-block-ip.cmd < test.json"`) rather than a
live pipe **worked perfectly** — logged the full sequence including a
successful `Blocked` message. This proved the script's read/parse/block
logic was correct; the hang was specific to *how* the real Wazuh agent
service invokes the script, not the reading method.

**Root cause:** `[Console]::In` relies on Windows console API handles.
The Wazuh agent runs as a Windows Service in **Session 0**, which has
no attached interactive console — under that context, direct
console-stream reads can hang indefinitely even though the underlying
pipe genuinely has data written to it. This is a known category of
Windows Service quirk, not specific to this script.

### Iteration 3 (final) — capture stdin to a temp file, read as plain file I/O

Rebuilt the `.cmd` wrapper to capture stdin into a temp file first
(`findstr "^" > tmpfile`, which handles piped input reliably regardless
of console context), then pass that file's path to the PowerShell
script via a `-InputFile` parameter. The script reads it with
`Get-Content -Raw` — plain file I/O, no dependency on console handles
at all. This is the version that's actually deployed and confirmed
working (see below).

**Bonus fix while rebuilding:** switched firewall operations from the
`New-NetFirewallRule`/`Remove-NetFirewallRule` PowerShell cmdlets to
calling the real `netsh.exe advfirewall firewall` commands directly.
The cmdlets go through a COM/WMI-backed provider with several seconds
of overhead per call (~7s measured to add a single rule during manual
testing) — with only a 30-second block window, that overhead alone
would eat a meaningful fraction of the block duration. Raw `netsh`
calls are the same mechanism already proven fast and reliable earlier
in this environment (the manual ICMP firewall rule from the networking
troubleshooting).

### Confirmed working, end-to-end

With the corrected script deployed and the detection rule itself fixed
(see the `if_matched_sid` correction in
`docs/02-detection-capabilities.md` — a separate, unrelated bug found
in parallel while retesting this), a real 25-request concurrent burst
from the RHEL VM produced the full chain:

```
Rule 100101 fires repeatedly → Rule 100102 escalates (level 10) →
Active Response 657 triggers (custom-block-ip.cmd - add) →
custom-block-ip.cmd: Blocked 192.168.0.201 (rule: Wazuh-Block-192.168.0.201)
```

visible in both `active-responses.log` and the Wazuh dashboard within
the same second. The block was confirmed to actually apply (not just
log) via a failed `curl` from the source IP during the block window,
and the automatic 30-second unblock was confirmed via Wazuh's native
`"command":"delete"` callback — no manual intervention needed.

**Scalability caveat still applies** — see above. This remains a
single-agent lab fix, not a production distribution solution, and is
documented as such.
