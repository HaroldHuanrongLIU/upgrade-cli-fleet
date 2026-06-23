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
