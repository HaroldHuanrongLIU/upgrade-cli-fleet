---
name: upgrade-cli-fleet
description: Use when the user wants to inventory, classify, plan, dry-run, or apply safe upgrades for many local CLI tools at once, especially macOS PATH fleets, package-manager-owned commands, owner-aware CLI upgrade plans, unknown command review, or non-shell argv-only upgrade recipes.
---

# Upgrade CLI Fleet

## Overview

Plan CLI upgrades by owner, not by guessed command-specific update commands. Treat local path and package-manager evidence as authoritative, keep unknown commands in review, and only apply low-risk argv actions after the user approves the plan.

## Workflow

1. Inventory the active `PATH` without executing discovered binaries.
   Record command name, path, canonical path, symlink status, duplicates, and PATH precedence. Do not probe unknown commands with `--help`, `--version`, or self-update flags.

2. Resolve ownership from local evidence.
   Prefer package-manager metadata and install paths over web identity. Safe manager metadata commands are acceptable when they query the manager itself, not arbitrary discovered binaries.

3. Classify each command.
   Use `primary`, `dependency`, `system`, `review`, or `unknown`. Hide dependency/system noise in summaries, but preserve it in audit details when the user asks for complete inventory.

4. Build an upgrade plan.
   Prefer manager-level upgrades over command-specific self-updaters. Put ambiguous, shadowed, conflicting, sudo-requiring, shell-based, or high-risk commands into review instead of inventing recipes.

5. Safety-gate every action.
   Accept only low-risk argv arrays. Reject shell strings, shell wrappers, `sudo`, `eval`, `source`, and recipes that depend on project context. Re-check the active command path before applying.

6. Present before applying.
   Show trusted actions, dry-run commands, skipped commands, review items, conflicts, unknowns, and evidence. Execute upgrades only after explicit user approval of the final plan.

## Reference Routing

- Read `references/safety-model.md` before proposing or applying any executable action.
- Read `references/manager-catalog.md` when resolving owners, choosing upgrade scope, or deciding whether a manager is trusted, skipped, or review-only.
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
