# LLM and Web Enrichment

Use enrichment only after local filtering decides which commands are visible non-dependency commands or when the user asks to explain a specific command.

## Boundary

- Local evidence is authoritative for path, canonical path, symlink status, PATH order, and dependency/system filtering.
- Web search can identify a public project, current install instructions, or current upgrade docs, but it cannot prove how the local binary was installed.
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
  "owner_manager": "homebrew | npm-global | pipx | uv-tool | mise | rustup | cargo | go | app-bundle | macos-system | unknown",
  "owner_package": "string or null",
  "visibility": "primary | dependency | system | review | unknown",
  "visibility_reason": "short local-evidence reason",
  "sources": [{"title": "string or null", "url": "string or null", "note": "string or null"}],
  "upgrade_recipe": {
    "title": "string",
    "argv": ["program", "arg"],
    "risk": "low",
    "requires_sudo": false,
    "uses_shell": false,
    "sources": [],
    "freshness_ttl_days": 14
  },
  "trust_decision": "trusted | review | blocked"
}
```

Use `null` for `upgrade_recipe` when a safe low-risk argv recipe cannot be justified.

## Recipe Rules

- Return argv arrays only.
- Do not return shell command strings.
- Do not use shell wrappers or `sudo`.
- Prefer owner/package-manager recipes over command-specific self-updaters.
- If the command appears manager-owned locally, say the owning manager should handle upgrades.
- Mark risk as `low` only for well-known, bounded upgrade commands.
- Mark risk as `medium` or `high` when the recipe may update an environment, modify many packages, require elevated permissions, or depend on project context.
- Keep freshness TTL short when a recipe may have changed recently.
