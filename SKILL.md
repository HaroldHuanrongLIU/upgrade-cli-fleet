---
name: upgrade-cli-fleet
description: Use when the user wants to inventory, classify, plan, dry-run, or apply safe upgrades for many local CLI tools at once, especially macOS PATH fleets, package-manager-owned commands, owner-aware CLI upgrade plans, unknown command review, or non-shell argv-only upgrade recipes.
---

# Upgrade CLI Fleet

## Overview

Plan CLI upgrades by owner, not by guessed command-specific update commands. Treat local path and package-manager evidence as authoritative, keep unknown commands in review, and only apply low-risk argv actions after the user approves the plan.

## Workflow

1. Inventory the active `PATH` without executing discovered binaries.
   Record command name, path, canonical path, symlink status, duplicates, and PATH precedence. Enumerate every npm global prefix (nvm node versions, `~/.npm-global`, `~/.local/lib/node_modules`), not just the active npm. Do not probe unknown commands with `--help`, `--version`, or self-update flags.

2. Resolve ownership from local evidence.
   Prefer package-manager metadata and install paths over web identity. Resolve the actual argv[0] name on PATH (it may differ from the canonical name, e.g., cursor-agent installed as `agent`). Check the binary's owner and writability — root-owned or non-writable paths need `sudo` and are blocked. Safe manager metadata commands are acceptable when they query the manager itself, not arbitrary discovered binaries.

3. Classify each command.
   Use `primary`, `dependency`, `system`, `review`, or `unknown`. Bucket each as upgrade / skip / no-upgrade-needed / cannot-upgrade per `references/classification-guide.md`. Hide dependency/system noise in summaries, but preserve it in audit details when the user asks for complete inventory.

4. Build an upgrade plan.
   Prefer manager-level upgrades over command-specific self-updaters. Put ambiguous, shadowed, conflicting, sudo-requiring, shell-based, or high-risk commands into review instead of inventing recipes.

5. Safety-gate every action.
   Accept only low-risk argv arrays. Reject shell strings, shell wrappers, `sudo`, `eval`, `source`, and recipes that depend on project context. Re-check the active command path before applying.

6. Collect current and target versions for every planned upgrade.
   Before asking for approval, run safe manager metadata commands (`brew outdated`, `npm outdated -g`, etc.) and bounded self-updater check commands to record the current version and the version the action will install. Do not probe unknown binaries with `--version`; use the manager or the tool's own bounded update subcommand output.

7. Present before applying.
   Build one consolidated plan. Put every Bucket A action into a single batch approval table with command name, current version, target version, owner, argv action, risk, and evidence. Summarize skipped, already-current, manual-only, and review items; do not ask about each one individually. Execute upgrades only after explicit user approval of the final action set.

8. Provide a post-apply summary.
   After upgrades finish, list what was upgraded (with before/after versions), what was already current, what was skipped or blocked, and any errors. Let the user audit the outcome without re-running the inventory.

## Defaults & Batching

Apply these defaults to reduce back-and-forth without weakening the safety model:

- **Skip by default.** Do not ask about system paths, app-bundle shims, language runtimes, dependency/non-CLI helpers, or review-only sources (conda, MacPorts, Nix, SDKMAN!, `~/go/bin`, `~/.cargo/bin`, pnpm standalone). Summarize them in the plan.
- **Manual-only by default.** Standalone self-updaters with unverified non-interactive flags, or known interactive TUIs, go into the manual-only table. List the bounded argv so the user can run it in their own terminal, but do not execute it automatically.
- **Batch approval.** Present every Bucket A action in one table and ask for one final approval. Do not ask per command.
- **Review bucket is narrow.** Only ambiguous ownership, PATH conflicts/shadowing, root-owned paths that need sudo, or shell-based recipes require a user decision.

## Reference Routing

- Read `references/safety-model.md` before proposing or applying any executable action.
- Read `references/manager-catalog.md` when resolving owners, choosing upgrade scope, or deciding whether a manager is trusted, skipped, or review-only.
- Read `references/classification-guide.md` when bucketing a discovered CLI into upgrade / skip / no-upgrade-needed / cannot-upgrade, or when deciding what to leave out of a plan.
- Read `references/llm-enrichment.md` when using web search, an LLM, or third-party docs to classify unknown commands or propose recipes.
- Read `references/plan-format.md` when the user asks for a structured plan, JSON-like output, dry-run report, or audit trail.

## Output Rules

Include enough evidence for the user to audit the plan:

- command name and active path
- resolved owner and package, or why ownership is unknown
- action as an argv array, or why no action is safe
- risk level and whether sudo or shell execution would be required
- conflicts, shadowed commands, and review-only commands
- commands that are skipped as dependencies or system tools

If the user says "upgrade everything" or similar, still plan first. Do not execute until they approve the concrete final actions.
