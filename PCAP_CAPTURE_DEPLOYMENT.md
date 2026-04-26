# Packet Capture (PCAP) — N2NHU Firewall v3.1

This release adds production-grade packet capture on top of the v3
IPv6/NAT64 firewall. Captures are declarative INI rules using the same
zone/address/service vocabulary as the policy and NAT engines, write
standards-compliant PcapNG with rule-id annotations on every frame, and
run on a bounded async write path that cannot block the dataplane.

## What you get

- **Declarative `capture.ini`.** Operators describe what to capture in
  the same language as policy and NAT. No special syntax, no shell
  filters, no BPF — captures are just rules.
- **Three capture points** at structural decision boundaries:
    - `ingress` — frame as it arrived on the wire (pre-policy, pre-NAT)
    - `post_policy` — frame after policy decision (allow or deny visible)
    - `post_nat` — frame after NAT rewrite (the bytes that are forwarded)
- **Rule-id annotations** on every captured frame as PcapNG comments:
  `capture_point=`, `decision=`, `rule_id=`, `nat_rule_id=`,
  `pbr_rule_id=`, `cap_rule=`. **Wireshark renders these in the Packet
  Comments tree natively.** This is the killer feature — no commercial
  firewall I know of does this well, and it follows directly from the
  declarative-rule architecture.
- **Bounded async write path.** Each pcap file gets its own writer
  thread and bounded queue. A slow file doesn't starve fast files; a
  full queue drops frames with a counter rather than blocking the
  dataplane.
- **Size-based rotation with retention.** `rotate_size_mb` triggers
  rotation; `rotate_keep` caps how many rotated files survive. Old
  files are pruned oldest-first.
- **Match filters.** Captures can be restricted by zone, address,
  service, and policy decision (`match_action = allow|deny|any`).
- **Per-rule snap_len.** Truncate to header only for high-volume flows;
  capture full payload for low-volume troubleshooting.
- **Offline CLI tool** (`n2nhu_pcap_tool`) — pure-Python pcapng reader
  with stats and filtered listing. No third-party dependencies.

## Quick start

```ini
# policies/capture.ini

[capture_order]
rules = cap_lan_egress, cap_denies

[cap_lan_egress]
capture_point = post_nat
src_zone = trusted
dst_zone = untrusted
src_addr = addr_lan_net
match_action = allow
pcap_file = /var/log/n2nhu/lan_egress.pcapng
rotate_size_mb = 100
rotate_keep = 5
snap_len = 1500

[cap_denies]
capture_point = post_policy
match_action = deny
pcap_file = /var/log/n2nhu/denies.pcapng
rotate_size_mb = 50
rotate_keep = 10
```

Then operate normally:

```python
from n2nhu_policy.router_shim import RouterShim
shim = RouterShim(config_root="/etc/n2nhu", interface_ips={...}, ...)
# capture starts automatically when capture.ini is present
shim.process_frame("lan", frame)  # captured per the rules
shim.shutdown()                   # flushes and closes pcap files
```

Inspect captures offline:

```
$ n2nhu_pcap_tool stats /var/log/n2nhu/lan_egress.pcapng
file:            /var/log/n2nhu/lan_egress.pcapng
frames:          14823
original bytes:  9847234
captured bytes:  9847234
duration:        3617.412s

By capture_point:
  post_nat             14823
By decision:
  allow                14823
By policy rule_id:
  p_lan_to_internet    14823
By NAT rule_id:
  n_snat_wan           14823

$ n2nhu_pcap_tool list capture.pcapng --match decision=deny --limit 10
#  103 t=1777231039.117  point=post_policy dec=deny  rule=p_block_blocked  ...
#  157 t=1777231042.812  point=post_policy dec=deny  rule=p_block_blocked  ...
...
```

## Configuration reference

### `[capture_order]`
- **rules** — comma-separated list of rule IDs in match-evaluation
  order. Empty means capture is disabled.

### Per-rule `[<rule_id>]`

| Key             | Type    | Default       | Description |
|-----------------|---------|---------------|-------------|
| `enabled`       | bool    | `true`        | Disable a rule without removing it. |
| `capture_point` | enum    | (required)    | `ingress` / `post_policy` / `post_nat` |
| `src_zone`      | string  | `any`         | Source zone or `any` |
| `dst_zone`      | string  | `any`         | Destination zone or `any` |
| `src_addr`      | string  | `addr_any`    | Address-object id (cross-validated) |
| `dst_addr`      | string  | `addr_any`    | Address-object id (cross-validated) |
| `service`       | string  | `svc_any`     | Service-object id (cross-validated) |
| `match_action`  | enum    | `any`         | `any` / `allow` / `deny` |
| `pcap_file`     | path    | (required)    | Output pcapng path (parent dir auto-created) |
| `rotate_size_mb`| int     | `0`           | Rotate at this many MB (0 = never) |
| `rotate_keep`   | int     | `0`           | Retain this many rotated files (0 = retain all) |
| `snap_len`      | int     | `0`           | Bytes captured per frame (0 = full frame) |
| `queue_depth`   | int     | `4096`        | Bounded async queue depth per file |

## Capture-point semantics

**INGRESS** captures the frame as it arrived from the wire. Egress
interface is not yet known, so capture rules with `dst_zone = any`
are the practical pattern. Use INGRESS to record everything entering
the firewall regardless of policy outcome — useful for forensic
investigation of incidents where you want to see what the firewall
saw before any decision was made.

**POST_POLICY** fires after the policy engine decides. Both allows and
denies trigger; combined with `match_action`, this is how you build
"deny log" captures. The frame bytes are still pre-NAT (policy runs
on pre-NAT addresses by SonicWall convention), so source addresses
are the original sender.

**POST_NAT** fires only on forwarded packets, after NAT rewrites
the frame. The captured bytes are exactly what will be transmitted
on the egress interface. Use POST_NAT to confirm what the WAN side
actually sees, including post-translation source ports.

A capture rule at POST_NAT will never fire for a denied packet — the
ordering is structural. The pipeline is:

```
ingress → INGRESS capture → policy → POST_POLICY capture
                                   ↓ (deny: drop)
                                   ↓ (allow)
                                   NAT → POST_NAT capture → forward
```

## What's not in this release

- **UI integration.** The capture configuration must be edited
  directly in `capture.ini`. UI v2.1 will add an editor and a "live
  captures" tab. Targeted for the next minor release.
- **Capture-on-demand.** All captures are persistent — they start
  when the shim starts and stop when it shuts down. Programmatic
  start/stop is available via the manager but not exposed as a
  config-driven feature in this release.
- **Time-based rotation.** Only size-based rotation is supported.
  `logrotate` and similar OS tools handle time-based rotation
  externally if needed.
- **Per-frame BPF filters.** The match vocabulary is the existing
  policy vocabulary; no raw BPF expressions. This is intentional —
  BPF in INI files would conflict with the algebraic-design thesis.

## Test coverage

102 capture-specific tests across six phases (K-P), all passing:

- **Phase K** (21 tests) — PcapNG writer format correctness, rotation,
  thread safety
- **Phase L** (23 tests) — `capture.ini` loader, defaults, validation,
  cross-reference checks
- **Phase M** (17 tests) — Capture engine: rule matching, zone/addr/
  service filters, action filter, point filter, pure-function property
- **Phase N** (16 tests) — Manager: writer pool, async queue, overflow
  drops, lifecycle, comment formatting, stats
- **Phase O** (11 tests) — RouterShim integration: end-to-end with
  policy + NAT, post-NAT bytes correctness, deny capture, multi-flow
- **Phase P** (14 tests) — CLI tool: stats command, list with filters,
  multi-match AND, error handling

Combined with the v3 IPv6 baseline, the suite now stands at **642 tests**
passing in approximately 13 seconds.

## Wireshark integration tip

The PcapNG comments are visible in Wireshark by default in the **Packet
Details** pane under "Packet comments". To use them in the column view:

1. **Edit → Preferences → Columns → Add**
2. Type: `Custom`, Field: `pkt_comment`

The annotation string then appears as a column, making it trivial to
sort or filter by `rule_id` or `decision` in the Wireshark UI.
