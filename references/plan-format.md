# Plan Format

Use this structure when the user asks for an upgrade plan, dry run, JSON-like audit output, or a final approval checklist.

## Human Summary

Report:

- total visible commands reviewed
- trusted actions planned
- review-only commands
- manual-only commands
- blocked commands
- skipped dependency/system commands
- conflicts or shadowed commands

## Action Rows

Each trusted action should include:

| Field | Meaning |
| --- | --- |
| `scope` | manager, package, or command being upgraded |
| `owner` | resolved manager and package, if known |
| `install_channel` | exact formula/cask, npm prefix, tool environment, app, or standalone updater channel |
| `current_version` | version currently installed locally |
| `target_version` | version the action will install |
| `target_source` | channel-matched manager metadata, non-mutating updater check, or official manifest used by that installer |
| `target_binding` | `pinned` when the argv encodes the version; otherwise `recheck-before-apply` with the residual race disclosed |
| `check_argv` | exact non-mutating argv used before approval; never the mutating updater; `null` only when an official channel API/manifest supplied the target without a subprocess |
| `argv` | exact argv array |
| `env` | fixed environment variables required for exact behavior, such as disabling manager auto-update |
| `risk` | usually `low`; otherwise do not auto-apply |
| `evidence` | local path/package evidence |
| `dry_run` | bounded dry-run equivalent, if available |
| `verification_argv` | exact bounded argv or manager query used after applying |

A dry run must not change installed software or package-manager installation state. It may write caches or temporary files only when those side effects are known and disclosed. If a command can implicitly update its manager, it is not a safe dry run unless that behavior is disabled.

## Review Rows

Each review item should include:

| Field | Meaning |
| --- | --- |
| `command` | command name |
| `active_path` | active PATH match |
| `reason` | why no trusted action was produced |
| `evidence` | path, manager, conflict, or source evidence |
| `next_step` | what the user should decide or inspect |

## Manual-Only Rows

Each manual-only item should include:

| Field | Meaning |
| --- | --- |
| `command` | command name |
| `current_version` | installed version, when safely known |
| `reason` | interaction, missing channel target, medium/high risk, or unavailable argv-native executor |
| `user_argv` | bounded argv the user may run in their own terminal |
| `target_evidence` | channel evidence if available; explicitly `unknown` otherwise |
| `verification_argv` | bounded command or manager query the user/agent should run afterward |

`user_argv` is a tokenized audit representation, not a shell command string. Do not claim it is directly pasteable and do not synthesize shell quoting; the user invokes the equivalent bounded command in their own terminal.

## Approval Prompt

Before applying, restate the exact action list:

```text
I will run these argv actions:
1. target=<approved-gh-version>, binding=recheck-before-apply,
   argv=["/opt/homebrew/bin/brew", "upgrade", "gh"], env={"HOMEBREW_NO_AUTO_UPDATE":"1"}
2. target=<approved-ruff-version>, binding=recheck-before-apply,
   argv=["pipx", "upgrade", "ruff"]

Review-only commands will not be modified.
```

Ask for approval only after the final action list, fixed environment variables, and channel-matched targets are concrete.

## Batch Approval

When the plan contains multiple Bucket A actions, present them as **one batch** and ask once:

```text
I will run these argv actions:
1. targets={<approved formula versions>}, binding=recheck-before-apply,
   argv=["/opt/homebrew/bin/brew", "upgrade", "ffmpeg", "fzf", "neovim"], env={"HOMEBREW_NO_AUTO_UPDATE":"1"}
2. target=<approved-codexbar-version>, binding=recheck-before-apply,
   argv=["/opt/homebrew/bin/brew", "upgrade", "--cask", "codexbar"], env={"HOMEBREW_NO_AUTO_UPDATE":"1"}
3. target=<approved-qwen-version>, binding=pinned,
   argv=["npm", "install", "-g", "@qwen-code/qwen-code@<approved-qwen-version>"]

Skipped, already-current, review-only, and manual-only commands will not be modified.
```

Do not ask for approval per action. The user responds to the whole batch.

## Post-Apply Summary

After running the approved actions, report a summary list:

| Section | Contents |
| --- | --- |
| **Upgraded** | command, owner/channel, previous version → verified target, argv used |
| **Already current** | command, owner, current version |
| **Partial / warnings** | target version confirmed, but a potentially relevant blocked lifecycle step or required health check leaves health uncertain |
| **Unverified** | verification ran but returned no definitive, parseable result |
| **Skipped** | command, reason (dependency, system, review-only source, user decision) |
| **Blocked / Cannot auto-upgrade** | command, reason (sudo required, curl-to-shell only, platform unsupported, error) |
| **Manual execution required** | command, reason (interactive TUI, missing target evidence, elevated risk, or no argv-native executor; user must run the bounded subcommand in their own terminal) |
| **Errors** | command, execution/verification stage, nonzero exit or target/path mismatch, truncated error message |
| **Remaining conflicts** | shadowed commands or duplicates that still need user decision |

Do not place an action in **Upgraded** based only on exit code 0 or updater output. Freshly re-resolve the command, verify its owner/channel, compare the observed version with the approved target, and require all planned bounded checks to pass. A non-health-affecting warning may be attached to Upgraded; a potentially relevant blocked script belongs in Partial / warnings. Keep the summary concise enough to scan, but include enough version and evidence detail that the user can audit the outcome without re-running the inventory.
