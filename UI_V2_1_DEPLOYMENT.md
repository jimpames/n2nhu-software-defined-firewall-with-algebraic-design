# UI v2.1 — N2NHU Firewall

This release brings the web UI to feature-parity with the engine. The
v2 UI could read every config file the engine accepted and edit
existing fields; v2.1 adds the missing primitive — **creating new
template instances, deleting them, and reordering them** — and brings
two new file types into the editable surface: **packet capture** and
**IPSec/strongSwan**.

## What's new in v2.1

### Schemas registered

| Schema | File | Sections |
|---|---|---|
| capture | `policies/capture.ini` | `[capture_order]` (fixed) + `cap*` (template) |
| ipsec   | `policies/ipsec.ini`   | `[ipsec_order]` (fixed) + `ipsec_proposal_*`, `ike_policy_*`, `ipsec_tunnel_*` (templates) |

These join the existing seven schemas (zones, addresses, services,
schedules, security, nat, pbr) for a total of nine. The dashboard
sidebar lists them in `order=` priority from each schema's
`[__schema__]` block, so capture and ipsec appear after pbr.

### Template instance management

Three new HTTP endpoints, each guarded by `ALLOW_EDIT`:

| Method | Path | Purpose |
|---|---|---|
| POST | `/ini/<n>/instance/add` | Create a new template instance with schema defaults |
| POST | `/ini/<n>/instance/delete` | Remove a template instance and strip from order list |
| POST | `/ini/<n>/instance/reorder` | Reorder instances via the `*_order` CSV field |

Each route writes the modified INI atomically and creates a git commit
through the existing `commit_change` pipeline. Failed validation
returns a flash message and redirects back to the edit page —
nothing partial is written.

### Edit-page UX

The edit page (`/ini/<n>/edit`) gains:

- **Per-instance Delete buttons** on every template-instance section
  header (with browser confirm dialog before submission)
- **"+ Add new …" cards** at the bottom of the page, one per
  template pattern in the schema. Each card has an instance-ID input
  with a regex pattern enforcing INI-safe characters, plus an
  emerald-styled Add button
- **Updated help banner** describing the new affordances

The Preview/Confirm pipeline for *editing existing fields* is unchanged
— add and delete commit immediately because they're structural
changes, while edits stay staged until the operator confirms a diff.

## Configuration semantics

### Adding a template instance

```
POST /ini/capture/instance/add
  pattern=cap*
  instance_id=cap_lan_egress
```

Behavior:

1. Validate pattern is a TEMPLATE section in the schema
2. Validate the instance ID is non-empty and contains only
   `[A-Za-z0-9_-]`
3. Canonicalize: if the operator typed `lan_egress` and the pattern
   prefix is `cap`, the new section is `[caplan_egress]` (the prefix
   is added). If they typed `cap_lan_egress` directly, that's used
   verbatim
4. Refuse if the section already exists
5. Build `[<new-name>]` with all schema defaults
6. Insert into the file at the position right after the last existing
   instance of the same template (keeping like with like)
7. If the schema has an order list (a fixed section with a
   csv_string field named `rules` or `tunnels`) AND the template
   pattern is one we expect to participate in evaluation order
   (`pbr*`, `p*`, `n*`, `cap*`, `ipsec_tunnel_*`), append the new ID
   to it. Reference-only templates (proposals, IKE policies,
   addresses, services) skip this step

### Deleting a template instance

```
POST /ini/capture/instance/delete
  instance=cap_lan_egress
```

Behavior:

1. Confirm the section exists and is a template instance (fixed
   sections cannot be deleted via this path)
2. Remove the section from the file
3. Strip the ID from any matching order list

### Reordering template instances

```
POST /ini/capture/instance/reorder
  pattern=cap*
  order=cap_b, cap_a, cap_c
```

Behavior:

1. Validate every ID in `order` exists in the file as a template
   instance matching the given pattern
2. Replace the order field's CSV value with the new list
3. The physical ordering of section blocks within the file is NOT
   reshuffled — operators read evaluation order from the `*_order`
   CSV. The block order in the file is cosmetic. (This matches how
   the engine reads the file.)

## Order-list discovery

The form_helpers module uses an explicit allow-list to decide which
template patterns participate in an order field:

```python
_ORDERED_PATTERNS = {
    "pbr*",            # PBR rules
    "p*",              # security policy rules
    "n*",              # NAT rules
    "cap*",            # capture rules
    "ipsec_tunnel_*",  # IPSec tunnels (proposals/ike_policies excluded)
}
```

This is necessary because the IPSec schema has a `[ipsec_order]`
section with a `tunnels` CSV — but proposals and IKE policies are
referenced by other sections, not evaluated in sequence. Without the
allow-list, adding a proposal would erroneously append it to the
tunnels list and produce a config the engine couldn't load.

## Test coverage

Phase V (21 tests) — schema additions and section-matching for
capture and IPSec.

Phase W (19 tests) — `add_template_instance` /
`delete_template_instance` / `reorder_template_instances` business
logic, including duplicate-ID rejection, invalid characters, empty
IDs, non-template patterns, unknown patterns, IPSec multi-template
behavior, and the proposals-don't-leak-into-tunnels guard.

Phase X (19 tests) — Flask route layer: redirects, file mutations,
flash messages, 404 for unknown schemas, 403 when readonly, GET
method-not-allowed.

Phase Y (12 tests) — end-to-end UI flows: edit page rendering with
new affordances, three-Add-form layout for IPSec, full add → edit
→ commit cycle, full IPSec setup persistence, add-then-delete
round-trip, reorder visibility through the edit view, all nine
schemas render cleanly.

Total v2.1 tests: **71**. Combined suite: **795 tests passing in
~33s**.

## What's NOT in v2.1

- **Drag-and-drop reordering.** The reorder endpoint exists and is
  fully tested, but the front end uses a plain CSV edit. A JavaScript
  drag-and-drop UI on top of the existing endpoint is straightforward
  for v2.2.
- **Live capture streaming.** Watching a `.pcapng` file as it grows
  and rendering frames in the UI would be a great feature but
  requires WebSocket/SSE plumbing that's a bigger lift than the rest
  of v2.1. Targeted for a "live ops" tab in v3.
- **Bulk import / diff-based config push.** Operators currently edit
  one file at a time. A bulk apply that takes a tarball or git ref
  and applies a multi-file change set in one transaction is on the
  v2.2 list.
- **Schema-driven validation hints.** Required fields are flagged in
  the schema but the UI's red asterisk style hasn't been added to the
  v2.1 templates. Cosmetic; on the polish list.

## Operator workflow examples

### Building an IPSec deployment from scratch

1. Open `/ini/ipsec/edit`
2. In the "+ Add new IPSec proposal" panel, type `default` → Add.
   This creates `[ipsec_proposal_default]` with schema defaults
3. In "+ Add new IKE policy", type `branch1` → Add. Creates
   `[ike_policy_branch1]`
4. In "+ Add new IPSec tunnel", type `branch1` → Add. Creates
   `[ipsec_tunnel_branch1]` AND adds it to `[ipsec_order]` tunnels
5. Now edit each section's fields (encryption suite for the proposal,
   peer addresses for the policy, subnet selectors for the tunnel)
   via the existing field editors. Click Preview to see the diff,
   then Confirm to commit
6. From a shell, run `n2nhu_ipsec_tool apply` to push to strongSwan

### Adding a packet capture

1. Open `/ini/capture/edit`
2. In "+ Add new Capture rule", type `lan_egress` → Add. Creates
   `[caplan_egress]` (note: the schema's prefix is `cap`, not
   `cap_`, so the operator should type `cap_lan_egress` to get
   the conventional name)
3. Edit the fields: capture_point=post_nat, src_zone=trusted,
   pcap_file=`/var/log/n2nhu/lan.pcapng`, etc.
4. Preview → Confirm. Capture starts on next RouterShim reload.

### Cleaning up a stale rule

1. Open the edit page for the relevant schema
2. Click 🗑 Delete on the rule's section header
3. Confirm in the browser dialog
4. Section is removed AND the ID is stripped from the order list
   automatically. A git commit is created with the message
   `UI: deleted instance <id> from <filename>`.
