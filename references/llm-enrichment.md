# LLM and Web Enrichment

Use enrichment only after local filtering decides which commands are visible non-dependency commands or when the user asks to explain a specific command.

## Boundary

- Local evidence is authoritative for path, canonical path, symlink status, PATH order, and dependency/system filtering.
- Web search can identify a public project, current install instructions, or current upgrade docs, but it cannot prove how the local binary was installed.
- A release version from GitHub, npm, Homebrew, an app store, or another channel cannot be used as the target unless local evidence ties the installed CLI to that same channel or to the updater manifest that consumes it.
- LLM output can classify and suggest recipes, but every executable recipe must pass `references/safety-model.md`.
- If ownership or upgrade scope is ambiguous, set the command to `review` or `unknown` and omit executable recipes.
- Prefer official documentation for current recipe checks. If recipe freshness matters, verify it before planning.

## Classification Shape

Use this shape for enriched classifications:

```json
{
  "command": "string",
  "category": "package-manager | language-toolchain | cloud-cli | container-cli | ai-coding-cli | app-bundle-shim | system-helper | unknown-review",
  "display_name": "string or null",
  "summary": "string or null",
  "confidence": 0.0,
  "owner_manager": "homebrew | npm-global | pipx | uv-tool | mise | rustup | standalone | cargo | go | app-bundle | macos-system | unknown",
  "owner_package": "string or null",
  "install_channel": "string or null",
  "visibility": "primary | dependency | system | review | unknown",
  "visibility_reason": "short local-evidence reason",
  "sources": [{"title": "string or null", "url": "string or null", "note": "string or null"}],
  "upgrade_recipe": {
    "title": "string",
    "argv": ["program", "arg"],
    "target_version": "string or null",
    "target_source": {"title": "string or null", "url": "string or null", "note": "string or null"},
    "target_binding": "pinned | recheck-before-apply | unknown",
    "check_argv": null,
    "verification_argv": ["program", "--version"],
    "env": {},
    "requires_argv_native_executor": true,
    "risk": "low",
    "requires_sudo": false,
    "uses_shell": false,
    "sources": [],
    "freshness_ttl_days": 14
  },
  "trust_decision": "candidate | manual-only | review | blocked"
}
```

Use `null` for `upgrade_recipe` when no bounded argv recipe can be justified. Enrichment may produce only a `candidate`, never a final Bucket A trust decision. A bounded recipe may have `trust_decision: "manual-only"` when target, risk, interaction, or executor requirements prevent automatic execution.

## Recipe Rules

- Return argv arrays only.
- Do not return shell command strings.
- Do not use shell wrappers or `sudo`.
- Prefer owner/package-manager recipes over command-specific self-updaters.
- If the command appears manager-owned locally, say the owning manager should handle upgrades.
- Require channel-matched target evidence before promoting an enriched candidate into an approval plan; otherwise keep it review or manual-only. Final Bucket A trust is decided only after local plan-stage checks.
- Mark risk as `low` only for well-known, bounded upgrade commands.
- Mark risk as `medium` or `high` when the recipe may update an environment, modify many packages, require elevated permissions, or depend on project context.
- Keep freshness TTL short when a recipe may have changed recently.
