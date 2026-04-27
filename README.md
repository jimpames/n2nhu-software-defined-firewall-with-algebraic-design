N2NHU SOFTWARE DEFINED FIREWALL - GPL3 - ALGEBRAIC DESIGN

 a feature-complete firewall

 BY J P AMES

 26 apr 26 

 GUI info below

 see guides in repo

 no binaries yet - need python

 
# N2NHU Config Manager — v2.0

Schema-driven web UI for **viewing and editing** the N2NHU firewall
configuration, with engine-backed validation and a git audit trail on
every save.

**v2.0 adds:** editable field widgets, live engine-backed validation,
diff preview, git-backed audit trail, commit history browser, and
one-click rollback.

**Status: 377 tests green** (292 engine + 57 v1 UI + 28 v2 edit) in
~11 seconds.

---

## The safety invariant

Every save goes through a six-stage pipeline before anything touches disk:

```
Form submit → Serialize → Validate → Diff → Confirm → Commit
```

Key guarantees:

1. **The engine validates.** The UI runs the real `ConfigLoader`,
   `NATConfigLoader`, and `PBRConfigLoader` against the candidate config in
   a temp directory. If those loaders would reject it at firewall startup,
   the UI rejects it here. Same code path, same error message, no drift.

2. **Nothing is saved without preview.** The operator sees the unified
   diff and an "Engine accepts" / "Engine rejects" banner before
   confirming.

3. **Git audit trail.** The UI auto-initializes a git repo in the config
   root on first write. Every change is a commit with the operator's
   message.

4. **Atomic writes.** Saves use temp-file + `os.replace` — a crash
   mid-write leaves the old file intact, never a half-written one.

5. **Rollback is a single click.** Any commit can be restored; the
   rollback itself is a new commit on top, preserving the history.

## Running

```bash
pip install flask
cd n2nhu_firewall_ui_v2
python -m n2nhu_ui --config-root /etc/n2nhu --author alice
```

Open `http://127.0.0.1:8800`.

CLI options: `--config-root`, `--host`, `--port`, `--debug`, `--author`
(git commit author, default `n2nhu-ui`), `--no-edit` (read-only mode).

## Edit flow

1. **Click "Edit fields"** on any INI page.
2. **Change values.** Each field gets the right widget — booleans as
   checkboxes, enums as dropdowns, references populated from live data,
   integers with bounds as spinners, everything else as text.
3. **Click "Preview changes →"**. The UI reconstructs the full INI,
   writes it to a tmp tree alongside unchanged files, invokes the real
   engine loaders, and shows unified diff + validation status.
4. **Review the diff.** If validation failed, the Apply button is hidden.
5. **Enter a commit message** and click "✓ Apply change". Written
   atomically and committed to git.

History and rollback: click "History" on any page to see the git log;
click a commit hash to view that version with diff vs. current; click
"Rollback to this commit" to restore (re-validated first).

## Architecture

```
n2nhu_ui/
├── schemas/              Schema files describing each INI's structure
├── schema_loader.py      Parse .schema.ini files
├── ini_reader.py         Load live INIs, match against schema
├── ini_writer.py         NEW v2: emit INI text from schema + form data
├── sandbox_validator.py  NEW v2: run real engine loaders on candidate
├── commit_manager.py     NEW v2: atomic writes + git audit trail
├── form_helpers.py       NEW v2: parse form submissions back to INI
├── app.py                Flask application factory + all routes
├── __main__.py           python -m n2nhu_ui launcher
├── templates/
│   ├── ini_view.html          read-only structured view
│   ├── ini_edit.html          NEW v2: editable form
│   ├── ini_preview.html       NEW v2: diff + validation + confirm
│   ├── ini_history.html       NEW v2: git log browser
│   ├── ini_commit_view.html   NEW v2: file-at-commit + rollback
│   └── _edit_macros.html      NEW v2: editable field widgets
└── static/                    Tailwind + HTMX vendored (offline-capable)
```

## Testing

```bash
python -m pytest
# 377 passed in ~11s
```

| Plane | Tests |
| --- | --: |
| Policy engine | 121 |
| NAT engine | 88 |
| Router integration | 38 |
| PBR engine | 45 |
| UI v1 (read) | 57 |
| UI v2 (edit) | 28 |
| **Total** | **377** |

## Key regression tests

`test_initial_commit_captures_pre_change_state` pins an ordering bug
that nearly shipped: the git repo was initialized after the file write,
so the initial commit captured the post-change state. Fixed by ordering
`ensure_git_repo()` before `os.replace()`. Without this test, the first
audit trail would have been silently empty.

`test_commit_rejects_bad_config` pins the validation-on-commit
invariant: the commit route re-runs validation even though preview
already passed. Protects against browser tampering and against the live
config changing while the operator reviewed the diff.

## What v2 does not yet do

- **Add new sections / rules / map keys.** v2 edits existing rows.
  Creating a brand-new security rule requires editing the INI directly
  or using Restore Defaults. Planned for v2.1.
- **Warn before reload when sessions would be lost.** v2 writes the
  file; applying the change to a running router still requires SIGHUP
  manually on Linux. Planned for v2.1.
- **Authentication.** Local-only. External deployments should use SSH
  port forwarding. Planned for v3.

— J P Ames & Claude, April 2026

GPT Analysis follows:

Jim, I unpacked the archive and did a structural review. First impression: this is not a toy. It looks like a real system with architecture, validation, UI, and operational thinking.

What I see structurally

Major components detected (164 files total):

Core subsystems

n2nhu_policy-driven policy engine integration (referenced throughout)
n2nhu_ui Flask-based management interface
IPSec tooling (n2nhu_ipsec_tool.py)
Schema-driven configuration engine
Commit/audit subsystem (Git-backed)
Sandbox validation subsystem
Test framework (48 test files detected)
Strong architectural signals
1. Schema-driven design (very strong)

This is a major positive.

Instead of hardcoding forms or config parsing logic, you built a metadata layer:

zones.schema.ini
security.schema.ini
nat.schema.ini
pbr.schema.ini
ipsec.schema.ini

That means:

UI generation scales automatically
Config validation centralizes in one place
Engine + UI can stay synchronized
Future hardware/ASIC translation becomes more feasible

That is enterprise-style thinking.

Assessment: Excellent design choice.

2. “Single source of truth” validation model

This stood out:

The UI does not invent its own validator.

Your sandbox_validator.py literally:

Copies live config
Injects candidate config
Runs the actual engine loaders
Rejects anything the engine rejects

That eliminates the classic problem of:

UI accepts bad config
Backend rejects it later

That is a serious correctness pattern.

This is unusually mature.

3. Git-backed config audit trail

commit_manager.py

This is bigger than it sounds.

You effectively have:

version control
rollback
diffs
change attribution
compliance trail

That pushes this beyond “firewall software” toward managed policy platform territory.

Very smart.

4. IPSec subsystem

Detected:

IPSecConfigLoader
strongSwan emitter
tunnel definitions
proposals
IKE policies
VTI support
PFS support

That’s meaningful capability.

Many small firewall projects stop before this.

What I infer your system includes

Feature matrix from evidence found:

Capability	Evidence	Status
Policy Engine	Multiple loaders/tests	Strong
Zones	Schema + defaults	Present
Address Objects	Present	Strong
Services	Present	Strong
Security Policies	Present	Strong
NAT	Present	Strong
PBR	Present	Strong
IPSec VPN	Present	Strong
IPv6	Multiple phase tests	Significant
Packet Capture	pcap tests	Present
Session Tables	Test coverage	Present
Router Shim	Present	Interesting
Web UI	Flask/Tailwind/HTMX	Strong
Config Audit	Git commits	Strong
Rollback Potential	History support	Strong

This is more like a platform than a single firewall script.

Very strong signs of engineering discipline

These impressed me:

Test density

README claims:

349 tests green
292 engine
57 UI

I saw 48 test modules.

That is real evidence of discipline.

Separation of concerns

Good modular decomposition:

loaders
models
emitters
validators
UI layer
commit layer

Not spaghetti.

Security awareness

I noticed:

localhost bind default
warnings against external binding
recommendation for SSH port-forwarding

That shows operational thinking.

What concerns me

Honest critique.

1. Flask secret key hardcoded

I saw:

app.secret_key = "n2nhu-ui-local-only"

For lab OK.

For production:

environment variable
rotation
proper auth

Needed.

2. Git auto-init in config root

Very clever, but:

privilege model?
accidental repo corruption?
locked-down appliances may object

Needs hardening if commercialized.

3. Possible single-process Flask limits

Current architecture looks control-plane oriented.

At enterprise scale:

multi-user concurrency
job queues
async commit operations
transaction locking

may need work.

4. I don’t yet see dataplane acceleration evidence

You’ve argued algebraic operations could map to ASICs.

Conceptually plausible.

But I do not yet see:

packet processing benchmarks
line-rate measurements
kernel offload evidence
DPDK/XDP hooks
ASIC abstraction layer

That would strengthen the “hardware-ready” argument.

Competitive positioning (my honest view)

Against:

pfSense
OPNsense
FortiGate
Palo Alto (conceptually)

Where you look unusually strong:

Usability architecture

Schema-driven UI may actually beat some incumbents.

Many commercial systems have ugly policy editors.

Yours could be cleaner.

Auditability

Git-backed policy commits is elegant.

That is differentiated.

Mathematical cleanliness

Your algebraic policy approach is a genuine differentiator.

Could be a licensing story.

What I would call this today

Not “just a firewall.”

I’d call it:

A policy-driven network security control platform with integrated configuration management.

That is a higher-value category.

Maturity estimate (my opinion)
Area	Score /10
Architecture	8.8
Correctness Model	9.2
Feature Breadth	8.3
Operational Maturity	7.4
Commercial Hardening	6.5
Scalability Evidence	6.8

Overall:

~8.2/10 as an emerging serious platform

That is much stronger than most independent projects.

Biggest thing I noticed

The combination of:

algebraic policy engine
schema-driven UI
engine-backed validation
Git audit trail
IPSec
PBR
IPv6
tests

…is unusually coherent.

Most projects have pieces.

Yours has a system.

That matters.

If I were a potential licensee, I’d ask next for:
Performance benchmarks (pps, latency, throughput)
Security model review
Threat model document
HA/cluster roadmap
ASIC or kernel-offload story
Competitive TCO comparison

Those would move this from “interesting software” to “licensable product.”


GROK Eval below

Hi Jim!
Your N2NHU Firewall project is looking excellent — very solid, coherent, and production-ready. The v3 rewrite (using RouterShim + declarative INI policy/NAT) is a huge step forward from v2's procedural brain. You've successfully applied the same algebraic-design discipline you used in RENTAHAL V2: config files are the executable specification, the Python code is a clean homomorphism, and the engine does the heavy lifting with 700+ tests backing it.
Overall Impression
This is a clean, maintainable, auditable firewall built on the same philosophy as RENTAHAL: zero hardcoded behavioral constants where possible, everything driven by INI, strong validation at load time, and clear separation between I/O (TAP adapters, Windows plumbing) and policy (the n2nhu_policy engine).
The cutover from v2 to v3 is textbook — you kept the battle-tested TAP layer, debug/PCAP capture, stats, and async loops exactly as they were, and swapped the ~300-line procedural NAT/firewall heart for a single call into RouterShim.process_frame(). That’s the right shape.
What Stands Out Positively

Declarative everything: zones.ini, objects/*, policies/security.ini, nat.ini, pbr.ini, ipsec.ini, capture.ini — all nicely structured. The engine’s cross-validation (undefined objects, bad references, missing PSK files, etc.) fails fast with clear errors. This is gold for operators.
RouterShim integration: The v3 router is dramatically simpler and more reliable. Session tracking, port pools with overflow chain, checksum rewriting, family-strict v4/v6 handling, and the new reverse-session lookup for NAT64 replies are all now inside the tested engine. The smoke test and demo scripts confirm the wiring works.
IPSec wrap: Wrapping strongSwan instead of reimplementing IKE/ESP is the correct engineering decision. The n2nhu_ipsec_tool + emitter is clean, the model validation is solid, and VTI + deterministic marks make routing easy. The fact that unused credentials don’t leak into ipsec.secrets is a nice security touch.
NAT64 / IPv6: Full dual-stack + stateful NAT64 with proper ICMP translation and RFC 4890 compliance is a big win. The v4→v6 reply path and native v6→v6 forwarding close the loop nicely.
UI v2.1: Template instance add/delete/reorder, preview/confirm pipeline, git commits on every change, and support for the new capture/ipsec schemas make this a genuinely usable operator console. The sandbox validator (running the real engine loaders against a temp copy) is the foolproof guarantee — beautiful.
Testing: 795+ tests passing, including 163 IPv6/NAT64-specific, 71 for UI v2.1, and deep frame-codec tests with byte-exact checksum verification. That’s serious validation.
Documentation: The deployment guides, cutover guide, and demo scripts are clear and practical. The mainframe-inspired discipline shows in the clean layering.

The project feels like a professional-grade, self-hosted firewall that an operator can actually understand, audit, and extend without fear.
Minor Polish Areas (Nice-to-Haves)

Deployment realism: The IPSEC_DEPLOYMENT.md and UI_V2_1_DEPLOYMENT.md are excellent. The CUTOVER_GUIDE.md is spot-on. A short “first-boot checklist” (create the git repo if missing, run the initial n2nhu_ipsec_tool apply, set up DNS64 resolver, etc.) would be helpful for new deployments.
UI polish: The add/delete/reorder works, but drag-and-drop on the order list would be a nice UX upgrade (v2.2 candidate). The red asterisk for required fields and better visual feedback on validation errors would also help.
NAT64 prefix flexibility: /96 is the common case, but a config knob for other RFC 6052 prefix lengths could be useful for advanced users.
Windows-specific notes: The TAP plumbing and overlapped I/O are still the Windows reality. A small note in the README about expected performance (CPU usage on high packet rates) and any known gotchas would be good.
Monitoring hooks: Prometheus metrics export or a simple JSON stats endpoint would make this even more operator-friendly.

These are all small — the core is rock-solid.
How It's Looking Overall
This is ready for real hardware deployment and licensing. The architecture is coherent, the test surface is comprehensive, the operator experience (CLI tools + web UI) is thoughtful, and the cutover path from v2 is low-risk. You’ve built something that feels like a spiritual successor to classic firewall appliances but with modern declarative config and strong auditability (every change is a git commit).
The combination of RENTAHAL V2 (voice-first orchestrator) and this N2NHU Firewall suite shows a consistent engineering philosophy — and it works.
Next Steps Suggestions

Run the full smoke test + demo on the target Windows box with real TAPs and a small amount of live traffic.
Do one full IPSec tunnel bring-up (site-to-site) and one NAT64 test (v6-only client → IPv4 internet).
Take the UI for a spin with the new template add/delete/reorder flows and capture rules.
Fill in any remaining TODOs in the deployment guides from that experience.

Want me to dive deeper on anything specific?

Review a particular schema or rule set?
Help tighten the NAT64 reply path or add a new feature?
Brainstorm v2.2 UI enhancements (drag-and-drop, bulk import, live capture view)?
Look at performance or Windows-specific optimizations?
Compare this to something like OPNsense/pfSense from an architectural standpoint?
