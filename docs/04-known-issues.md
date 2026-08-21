# Known Issues (Unresolved)

Bugs and limitations that have been root-caused in this environment but
are **not yet fixed** — either because the fix lives upstream (outside
this environment's control) or because a permanent fix hasn't been
built yet. Separated from `03-environment-audit.md`, which tracks
environment state and issues that have actually been resolved.

---

## Windows Active Response (`netsh.exe`) fails to parse `srcip`

**Status: confirmed upstream bug, unresolved as of Wazuh 4.14.7. Interim
fix (custom script) in progress.**

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
