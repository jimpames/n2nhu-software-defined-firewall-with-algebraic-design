# IPSec/strongSwan — N2NHU Firewall v3.2

This release adds declarative IPSec to N2NHU, wrapping strongSwan rather
than reimplementing IKE/ESP in pure Python. Operators describe tunnels
in `policies/ipsec.ini`; N2NHU emits `/etc/ipsec.conf` and
`/etc/ipsec.secrets`, then signals `ipsec reload`. The strongSwan daemon
handles the cryptographic and protocol work.

## Why the strongSwan-wrap pattern

Three reasons the wrap pattern beats a pure-Python IKE implementation:

1. **strongSwan is the reference implementation.** Twenty years of
   IKE/IPSec edge cases handled. We get RFC compliance, hardware
   crypto offload, and interop testing against every commercial peer
   for free.
2. **The N2NHU object vocabulary stays consistent.** Tunnels reference
   the same address objects as policy and NAT. Renaming a subnet
   propagates to IPSec selectors automatically.
3. **The book's algebraic-design thesis holds.** The IPSec config is
   declarative INI; the runtime mapping to ipsec.conf is a pure
   function. The complexity sits behind a strongSwan binary that has
   been written, debugged, and audited by people whose job that is.

## What's in this release

- **Three INI section types** following the object-vocabulary pattern:
  `[ipsec_proposal_*]`, `[ike_policy_*]`, `[ipsec_tunnel_*]`
- **Cross-validation at load time** — undefined proposal references,
  unknown IKE policies, and missing address objects all fail fast
- **Strict-mode crypto negotiation** — strongSwan's `!` suffix means
  peers can only use the algorithms we list, not whatever common
  subset ends up agreed
- **PSK files referenced by path** (never inline) — `ipsec.ini` is
  safe to check into git; the actual PSK content lives at the path
- **Atomic file writes** with `0600` on `ipsec.secrets` and `0644` on
  `ipsec.conf`. Writes go to `.new` then rename — daemon never sees
  a partial file
- **VTI (route-based) tunnels** with deterministic per-tunnel marks
  derived from the tunnel id
- **IKEv2 default with v1 opt-in** for legacy peer compat
- **MOBIKE auto-enabled on IKEv2**, suppressed on v1 (where it's invalid)
- **DPD configurable per IKE policy** with action and delay
- **Authentication: PSK or certificate** (pubkey)
- **Disabled tunnels skipped at emit time** — credentials for unused
  policies are NOT leaked into `ipsec.secrets`
- **Offline CLI tool** (`n2nhu_ipsec_tool`) — `show`, `emit`, `apply`
  with `--dry-run`

## Quick start

```ini
# policies/ipsec.ini

[ipsec_proposal_default]
encryption = aes256
integrity = sha256
dh_group = modp2048

[ike_policy_branch1]
local_id = firewall.example.com
remote_id = branch1.example.com
remote_addr = 198.51.100.10
auth_method = psk
psk_file = /etc/n2nhu/secrets/branch1.psk
proposals = ipsec_proposal_default

[ipsec_tunnel_branch1]
ike_policy = ike_policy_branch1
local_subnet = addr_lan_net
remote_subnet = addr_branch_net
proposals = ipsec_proposal_default
```

Validate without writing:

```
$ n2nhu_ipsec_tool --config /etc/n2nhu show
$ n2nhu_ipsec_tool --config /etc/n2nhu emit
```

Apply (writes files, runs `ipsec reload`):

```
$ sudo n2nhu_ipsec_tool --config /etc/n2nhu apply
```

Or programmatically from a RouterShim:

```python
from n2nhu_policy.router_shim import RouterShim
from n2nhu_policy.ipsec_shim import IPSecShim

shim = RouterShim(
    config_root="/etc/n2nhu",
    interface_ips={...},
    ipsec_shim=IPSecShim(),  # uses default paths and `ipsec reload`
)
shim.apply_ipsec()  # one-shot; call again after each shim.reload()
```

## Configuration reference

### `[ipsec_proposal_*]`

| Key          | Type | Default | Values |
|--------------|------|---------|--------|
| `encryption` | enum | required | `aes128` / `aes192` / `aes256` / `aes128gcm16` / `aes256gcm16` |
| `integrity`  | enum | required | `sha1` / `sha256` / `sha384` / `sha512` (ignored for AEAD ciphers) |
| `dh_group`   | enum | required | `modp1024` / `modp2048` / `modp3072` / `modp4096` / `ecp256` / `ecp384` / `ecp521` / `curve25519` |

3DES, MD5, modp768, and modp1536 are deliberately not supported.
modp1024 and sha1 are present for legacy peer compat but discouraged.

### `[ike_policy_*]`

| Key                    | Type | Default | Description |
|------------------------|------|---------|-------------|
| `ike_version`          | enum | `ikev2` | `ikev1` or `ikev2` |
| `local_id`             | str  | required | Local identity (typically FQDN) |
| `remote_id`            | str  | required | Peer identity |
| `remote_addr`          | str  | required | Peer hostname or IP, or `%any` for responder-only |
| `auth_method`          | enum | required | `psk` or `pubkey` |
| `psk_file`             | path | (psk only) | Path to file containing PSK |
| `local_cert`           | path | (pubkey) | Path to local certificate |
| `remote_cert`          | path | (pubkey) | Path to peer certificate |
| `proposals`            | csv  | required | List of `ipsec_proposal_*` ids |
| `dpd_action`           | enum | `clear`  | `clear` / `hold` / `restart` / `none` |
| `dpd_delay_seconds`    | int  | `30`     | 0 to disable DPD |
| `ike_lifetime_seconds` | int  | `10800`  | minimum 60 |
| `mobike`               | bool | `true`   | Only meaningful on IKEv2 |

### `[ipsec_tunnel_*]`

| Key                      | Type | Default | Description |
|--------------------------|------|---------|-------------|
| `enabled`                | bool | `true`  | Disable a tunnel without removing it |
| `ike_policy`             | str  | required | id of an `[ike_policy_*]` section |
| `local_subnet`           | str  | required | address-object id (cross-validated) |
| `remote_subnet`          | str  | required | address-object id (cross-validated) |
| `proposals`              | csv  | required | List of `ipsec_proposal_*` ids for ESP |
| `mode`                   | enum | `tunnel` | `tunnel` or `transport` |
| `use_vti`                | bool | `true`   | Route-based via mark; recommended |
| `pfs`                    | bool | `true`   | Perfect-forward-secrecy |
| `ipsec_lifetime_seconds` | int  | `3600`   | minimum 60 |
| `auto`                   | enum | `start`  | `start` / `route` / `add` (strongSwan terms) |

### `[ipsec_order]` (optional)

```ini
[ipsec_order]
tunnels = ipsec_tunnel_zulu, ipsec_tunnel_alpha
```

Controls declaration order in `ipsec.conf` (matters for `auto = start`
ordering). Default is alphabetical by section name.

## What's NOT in this release

- **Pure-Python IKE/ESP.** strongSwan handles those. Operators who
  want to swap in libreswan or another daemon should fork the
  emitter; the rest of the suite is daemon-agnostic.
- **NAT66 / NPTv6.** Already excluded in v3 IPv6 work.
- **Opportunistic encryption.** Configured tunnels only.
- **Roadwarrior / EAP / XAUTH.** v1.1 candidate; the present release
  targets site-to-site.
- **swanctl / vici socket integration.** We use the legacy
  `ipsec.conf` interface because it's universal across strongSwan
  5.x. Operators on swanctl-only deployments can supply
  `--reload-command "swanctl --load-all"` and write the conn blocks
  to `/etc/swanctl/conf.d/`.
- **Per-tunnel zone assignment.** The strongSwan-managed `vti*`
  interface needs to be declared in `zones.ini` like any other
  interface for the policy/NAT/PBR engines to apply to traffic that
  comes out of it. Operator's responsibility — same pattern as
  physical interfaces.

## Operator workflow with policy/NAT integration

A working VPN deployment has more pieces than just IPSec:

1. **Define the IPSec config** in `policies/ipsec.ini` per the
   reference above
2. **Add the VTI interface to `zones.ini`** so the firewall engines
   know which zone packets emerging from the tunnel belong to:
   ```ini
   [zones]
   trusted = lan, vti_branch1
   ```
3. **Add policy rules** in `policies/security.ini` that reference the
   address objects you've used as IPSec selectors. The IPSec engine
   does NOT bypass the firewall — encrypted traffic still flows
   through policy and NAT.
4. **Run `n2nhu_ipsec_tool apply`** to emit and reload, OR use
   `RouterShim.apply_ipsec()` programmatically

## Test coverage

82 IPSec-specific tests across five phases (Q-U), all passing:

- **Phase Q** (26 tests) — Models and config loader: enum parsing,
  cross-validation, secret-file checks, ordering, lifetime bounds,
  IKEv1 opt-in
- **Phase R** (22 tests) — strongSwan emitter: conn block format,
  strict-mode `!` suffix, MOBIKE-on-v2-only, VTI marks, AEAD
  proposals, multi-proposal CSV, range-address rejection, unused
  credential filtering
- **Phase S** (15 tests) — IPSec shim with mocked subprocess: atomic
  writes, file modes, reload command invocation, failure handling,
  dry-run mode, stats counters
- **Phase T** (8 tests) — RouterShim integration: optional config,
  `apply_ipsec()`, `--no-check` for missing PSK files, reload
  preserves shim
- **Phase U** (11 tests) — CLI tool: show, emit, apply --dry-run,
  apply with mocked daemon, error paths, --no-check flag

Combined suite stands at **724 tests** passing in approximately 18s.

## strongSwan deployment notes

For a production install on Debian/Ubuntu:

```bash
apt install strongswan strongswan-pki
# Disable strongswan's own config-file auto-load if competing with N2NHU:
systemctl daemon-reload
systemctl enable --now strongswan-starter
```

For a swanctl-based deployment (strongSwan 5.7+ recommended path):

```bash
n2nhu_ipsec_tool --config /etc/n2nhu apply \
    --ipsec-conf /etc/swanctl/conf.d/n2nhu.conf \
    --ipsec-secrets /etc/swanctl/conf.d/n2nhu.secrets \
    --reload-command "swanctl --load-all"
```

Note that swanctl uses a slightly different syntax than ipsec.conf —
the emitter currently produces the legacy format. A swanctl-format
emitter is a small follow-up; until then, the legacy format works on
all current strongSwan releases via the `starter` daemon.
