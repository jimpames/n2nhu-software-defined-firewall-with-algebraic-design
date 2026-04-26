# N2NHU Firewall — IPv6 & Stateful NAT64 (v3)

This release adds full IPv6 support and stateful NAT64 (RFC 6146) to
the N2NHU Firewall Suite. It targets a 2026 internet where clients
increasingly run on IPv6-only networks while substantial legacy content
remains IPv4-only.

## What's in this release

**Dual-stack packet processing.** The firewall now parses, classifies,
and forwards both IPv4 and IPv6 frames through a single declarative
policy fabric. Rules are family-strict: a rule whose destination
address object holds only IPv4 addresses does not match IPv6 packets,
and vice versa. Address groups may mix families — e.g., an "internal
networks" group naturally spans both.

**Stateful NAT64** per RFC 6146, using RFC 6052 address embedding.
An IPv6 packet destined for a NAT64 prefix (`64:ff9b::/96` by default)
is translated into an IPv4 packet with a pool-allocated source port,
forwarded to the v4 server, and the reply is reverse-translated back
to the original v6 client.

**Cross-family ICMP translation** per RFC 7915 §5. Echo, Destination
Unreachable, Time Exceeded, and Packet Too Big all translate cleanly
in both directions, including the recursive translation of embedded
original packets inside ICMP errors.

**ICMPv6 RFC 4890 compliance.** Default services include a permit
group for Neighbor Discovery (133-136), Packet Too Big (2 — critical
for PMTU), Destination Unreachable (1), Time Exceeded (3), and
Multicast Listener Discovery (130-132, 143). Omitting these breaks
IPv6 operation; the shipped `services.ini` includes them by default.

**UI support.** NAT rules can select `nat64` as a fifth type in the
edit form, with the NAT64 prefix and v4 pool as new fields.

## Sample configuration — `policies/nat.ini`

    [nat_order]
    rules = nat64_primary

    [nat64_primary]
    type = nat64
    src_zone = trusted
    dst_zone = untrusted
    src_addr = addr_v6_clients
    dst_addr = addr_any_v6
    service = svc_any
    schedule = sched_always
    translate_src_to_v4 = addr_firewall_v4
    port_pool = 20000-60000

`addr_v6_clients` should be the v6 network operating as v6-only (e.g.
`2001:db8:cafe::/48`). `addr_firewall_v4` must be a host-type address
object pointing at the firewall's public IPv4 address. The
`nat64_prefix` field is optional and defaults to the well-known
`64:ff9b::/96`.

## DNS64 deployment

N2NHU does not ship a DNS resolver. A DNS64 resolver must run
alongside N2NHU to synthesize AAAA records that point into the NAT64
prefix, for hostnames that have only A records.

Recommended: Unbound. Configuration fragment for `unbound.conf`:

    server:
        module-config: "dns64 iterator"
        dns64-prefix: 64:ff9b::/96
        dns64-synthall: no

Clients on the v6-only segment should use this resolver (via Router
Advertisement RDNSS option, DHCPv6, or static config).

## Field-test plan

Before declaring a deployment operational, verify end-to-end:

1. **v6-only client acquires an address.** On the v6-only LAN, an
   Android phone (airplane mode, Wi-Fi on, v4 disabled at the AP) or a
   Linux VM with IPv4 disabled should obtain a v6 address via SLAAC
   from the router/firewall and a DNS64-capable resolver.

2. **AAAA synthesis works.** From the client, `nslookup example.org`
   should return a synthetic AAAA record within `64:ff9b::/96`. A
   hostname with real AAAA records (google.com) should return its
   native AAAA, not the synthesized one.

3. **Packet arrives at N2NHU as v6.** Confirm via `shim.stats`:
   `frames_in_v6` increments as the client initiates connections.

4. **Packet exits as v4.** `frames_in_v6` and `nat64_v6_to_v4`
   increment together; the session table shows a session with
   `orig_src_ip` as the v6 client and `xlated_src_ip` as the
   firewall's v4 address.

5. **Reply returns and reassembles as v6.** The client receives a
   v6 response; `nat64_v4_to_v6` increments. The reply traverses the
   firewall via the reverse-session lookup added in this revision —
   no policy re-evaluation; the session is the stateful permit.

6. **PMTU works.** Send a 1400-byte payload end-to-end. Any
   intermediate router generating a Packet-Too-Big ICMPv6 should
   be visible and the client should adapt.

## What's not in this release

- **NAT66 / NPTv6.** Deliberately not implemented — operator
  consensus treats v6 NAT as an antipattern.
- **NAT64 prefix lengths other than /96.** RFC 6052 permits /32, /40,
  /48, /56, /64 with different embedding layouts. v1 supports /96
  only, which covers the overwhelming majority of deployments.

## What's now complete (added in this revision)

- **v4→v6 reply path.** A v4 frame arriving at the firewall whose
  destination matches a NAT64 rule's pool address and whose 5-tuple
  matches an established session is reverse-translated v4→v6 and
  emitted on the original v6 client's ingress interface. Reply
  traffic on an established session is the stateful permit — policy
  is not re-evaluated, matching production firewall convention.
- **Native v6→v6 forwarding.** When the operator configures
  `interface_networks_v6` on the RouterShim, v6 frames whose
  destination does not match any NAT64 prefix are routed natively
  through the same policy/PBR pipeline as v4 (no rewrite, no
  checksum touch — v6 has no NAT in v1). Family-strict address
  objects ensure v4 rules never match v6 packets and vice versa.

## Test coverage

540 tests pass with zero skipped on the release branch. 163 of those
tests are IPv6/NAT64-specific, covering:

- Address family widening of config loader, resolver, and session table
- IPv6 frame codec with extension header chain walking
- ICMPv6 RFC 4890 classification
- RFC 6052 address embedding byte-exact against published examples
- RFC 7915 §5 ICMP/ICMPv6 cross-translation in both directions
- RouterShim v6→v4 integration with session creation
- RouterShim v4→v6 reply path with reverse session lookup
- Native v6→v6 forwarding with policy/PBR enforcement
- Backward compatibility: legacy v6-drop behavior preserved when no
  v6 routes are configured
- UI schema registration of the NAT64 type
