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
