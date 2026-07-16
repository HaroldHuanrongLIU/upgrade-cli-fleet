---
name: upgrade-cli-fleet
description: Use when the user wants to inventory, classify, plan, dry-run, or apply safe upgrades for many local CLI tools at once, especially macOS PATH fleets, package-manager-owned commands, owner-aware CLI upgrade plans, unknown command review, or non-shell argv-only upgrade recipes.
---

# Upgrade CLI Fleet

## Overview

Plan CLI upgrades by owner, not by guessed command-specific update commands. Treat local path and package-manager evidence as authoritative, keep unknown commands in review, make no installed-software changes before approval, and only apply low-risk argv actions that can be verified afterward.

## Workflow

1. Inventory the active `PATH` without executing discovered binaries.
   De-duplicate repeated PATH directories, then record command name, path, canonical path, symlink status, duplicates, and PATH precedence. Multiple names or symlinks that resolve to the same canonical target and owner are aliases, not conflicts unless their behavior or precedence differs. Enumerate every npm global prefix (nvm node versions, `~/.npm-global`, `~/.local/lib/node_modules`), not just the active npm. For very large fleets, keep the full audit but summarize system, TeX, Conda, and dependency-heavy paths by count so third-party CLI candidates remain visible. Do not probe unknown commands with `--help`, `--version`, or self-update flags.

2. Resolve ownership from local evidence.
   Prefer package-manager metadata and install paths over web identity. Resolve the actual argv[0] name on PATH (it may differ from the canonical name, e.g., cursor-agent installed as `agent`). Use filesystem owner/mode and the replacement target's parent directory to decide whether a self-update is locally writable. A sandbox `EPERM`, denied write probe, or restricted execution environment is not proof that the user lacks OS-level permission; report it separately and never infer a need for `sudo` or `chown` from the sandbox alone. Safe manager metadata commands are acceptable when they query the manager itself, not arbitrary discovered binaries.

3. Classify each command.
   Use `primary`, `dependency`, `system`, `review`, or `unknown`. Bucket each as upgrade / skip / no-upgrade-needed / cannot-auto-upgrade per `references/classification-guide.md`. Hide dependency/system noise in summaries, but preserve it in audit details when the user asks for complete inventory.

4. Build an upgrade plan.
   Prefer package-specific manager upgrades over command-specific self-updaters. Before approval, do not run an updater or any manager command that can change installed software or implicitly update the manager itself. Disable implicit manager auto-update for metadata checks where supported. Cache or temporary-file writes are acceptable only when they cannot change installed software and are disclosed in the plan. If a safe metadata query is unavailable, classify the item as manual-only or review instead of using the mutating updater as a check.

5. Safety-gate every action.
   Record every subprocess recipe as an exact argv array. Execute an approved upgrade action only through an API that preserves argv token boundaries; a tool that accepts only a shell command string is not an argv-native executor, so the action stays manual-only. Block shell-string diagnostics by default too. A known-manager read-only query is an exception only when the host explicitly guarantees no login/profile startup or shell expansion, and the command contains only fixed allowlisted environment assignments, the validated absolute manager path, and fixed literal arguments. Never interpolate package names, user input, or unknown data, and never use the exception for an updater. Reject shell recipes, shell wrappers, `sudo`, `eval`, `source`, and recipes that depend on project context. Re-check the active command path before applying.

6. Collect current and target versions for every planned upgrade.
   Before asking for approval, run only non-mutating manager metadata commands and bounded check/dry-run commands to record the current version and the version the action will install. Target evidence must match the local install channel: prefer the owning manager's metadata, then the standalone updater's documented check or release manifest. A GitHub release, npm tag, or other channel's latest version is supporting context, not proof of what this installation will receive. Bind the approved target into the argv when the manager supports a documented version pin; otherwise mark the action `recheck-before-apply` and repeat the same target query immediately before execution. Do not use the mutating update subcommand as a pre-approval check. If no safe channel-matched target evidence exists, keep the updater manual-only.

7. Present before applying.
   Build one consolidated plan. Put every Bucket A action into a single batch approval table with owner/channel, current and channel-matched target versions, target source/binding, argv and fixed environment, post-apply verification, risk, and evidence. Summarize skipped, already-current, manual-only, and review items; do not ask about each one individually. Execute upgrades only after explicit user approval of the final action set.

8. Provide a post-apply summary.
   After each action, freshly re-resolve the command path and verify the installed version through manager metadata or a documented bounded version/check argv. Exit code 0, download output, or checksum output alone does not prove the installation changed. Report upgraded, already current, partial/warning, unverified, skipped/blocked, and error outcomes separately so the user can audit the result without re-running the inventory.

## Defaults & Batching

Apply these defaults to reduce back-and-forth without weakening the safety model:

- **Skip by default.** Do not ask about system paths, app-bundle shims, language runtimes, dependency/non-CLI helpers, or review-only sources (conda, MacPorts, Nix, SDKMAN!, `~/go/bin`, `~/.cargo/bin`, pnpm standalone). Summarize them in the plan.
- **Manual-only by default.** Standalone self-updaters with unverified non-interactive flags, known interactive TUIs, no safe channel-matched target evidence, or no argv-native execution path go into the manual-only table. List the bounded argv so the user can run it in their own terminal, but do not execute it automatically.
- **Batch approval.** Present every Bucket A action in one table and ask for one final approval. Do not ask per command.
- **Review bucket is narrow.** Keep only ambiguous ownership, genuinely different PATH targets, or uncertainty about whether a recipe needs sudo/shell in review. Confirmed blocked and manual-only items are summarized without another question.

## Reference Routing

- Read `references/safety-model.md` before proposing or applying any executable action.
- Read `references/manager-catalog.md` when resolving owners, choosing upgrade scope, or deciding whether a manager is trusted, skipped, or review-only.
- Read `references/classification-guide.md` when bucketing a discovered CLI into upgrade / skip / no-upgrade-needed / cannot-auto-upgrade, or when deciding what to leave out of a plan.
- Read `references/llm-enrichment.md` when using web search, an LLM, or third-party docs to classify unknown commands or propose recipes.
- Read `references/plan-format.md` whenever producing an approval plan, applying actions, or reporting outcomes, and when the user asks for JSON-like output, a dry-run report, or an audit trail.

## Output Rules

Include enough evidence for the user to audit the plan:

- command name and active path
- resolved owner and package, or why ownership is unknown
- install channel, current/target versions, and the channel-matched target source
- action as an argv array, or why no action is safe
- pre-approval check and post-apply verification argv, when applicable
- risk level and whether sudo or shell execution would be required
- sandbox/executor limitations separately from real filesystem permissions
- conflicts, shadowed commands, and review-only commands
- commands that are skipped as dependencies or system tools

If the user says "upgrade everything" or similar, still plan first. Do not execute until they approve the concrete final actions.
