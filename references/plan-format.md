# Plan Format

Use this structure when the user asks for an upgrade plan, dry run, JSON-like audit output, or a final approval checklist.

## Human Summary

Report:

- total visible commands reviewed
- trusted actions planned
- review-only commands
- blocked commands
- skipped dependency/system commands
- conflicts or shadowed commands

## Action Rows

Each trusted action should include:

| Field | Meaning |
| --- | --- |
| `scope` | manager, package, or command being upgraded |
| `owner` | resolved manager and package, if known |
| `current_version` | version currently installed locally |
| `target_version` | version the action will install (from manager metadata or the tool's own update check) |
| `argv` | exact argv array |
| `risk` | usually `low`; otherwise do not auto-apply |
| `evidence` | local path/package evidence |
| `dry_run` | dry-run equivalent, if available |

## Review Rows

Each review item should include:

| Field | Meaning |
| --- | --- |
| `command` | command name |
| `active_path` | active PATH match |
| `reason` | why no trusted action was produced |
| `evidence` | path, manager, conflict, or source evidence |
| `next_step` | what the user should decide or inspect |

## Approval Prompt

Before applying, restate the exact action list:

```text
I will run these argv actions:
1. ["brew", "upgrade"]
2. ["pipx", "upgrade-all"]

Review-only commands will not be modified.
```

Ask for approval only after the final action list is concrete.

## Post-Apply Summary

After running the approved actions, report a summary list:

| Section | Contents |
| --- | --- |
| **Upgraded** | command, owner, previous version → new version, argv used |
| **Already current** | command, owner, current version |
| **Skipped** | command, reason (dependency, system, review-only source, user decision) |
| **Blocked / Cannot upgrade** | command, reason (sudo required, curl-to-shell only, platform unsupported, error) |
| **Manual execution required** | command, reason (interactive TUI that cannot be confirmed in non-interactive Bash; user must run the bounded subcommand in their own terminal) |
| **Errors** | command, exit code, truncated error message |
| **Remaining conflicts** | shadowed commands or duplicates that still need user decision |

Keep the summary concise enough to scan, but include enough version and evidence detail that the user can audit the outcome without re-running the inventory.
