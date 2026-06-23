# Safety Model

Use ownership resolution before upgrade execution:

```text
PATH inventory -> ownership resolution -> trusted plan -> safety gate -> apply
```

## Hard Rules

- Never execute unknown binaries.
- Never probe unknown commands with `--help`, `--version`, update flags, or installer flags.
- Treat local path, canonical path, symlink status, package metadata, and PATH order as objective evidence.
- Treat web pages and LLM output as supporting facts, not proof of local ownership.
- Keep macOS system paths skipped unless the user explicitly asks for audit-only inventory.
- Surface conflicts and shadowed commands for review.
- Prefer owner/package-manager upgrades over tool-specific self-updaters.
- Execute actions as argv arrays, not shell strings.
- Block `sudo` by default.
- Block shell wrappers such as `sh -c`, `bash -c`, `zsh -c`, `/usr/bin/env sh`, `/usr/bin/env bash`, and `/usr/bin/env zsh`.
- Block `eval`, `source`, curl-to-shell, and installer scripts.
- Treat `--yes` or equivalent confirmation flags as valid only for trusted low-risk actions.

## Blocked Recipe Shapes

Reject these shapes:

```text
sh -c "tool update"
bash -c "tool update"
zsh -c "tool update"
sudo tool update
/usr/bin/env sh -c "tool update"
eval "tool update"
source script.sh
curl https://example/install.sh | sh
```

## Accepted Recipe Shape

Accepted executable actions must be explicit argv arrays:

```json
["brew", "upgrade"]
```

```json
["uv", "tool", "upgrade", "ruff"]
```

## Apply Rules

Before applying:

1. Re-check that the program named in `argv[0]` resolves to the same trusted path used during planning.
2. Re-check that the action is still low-risk, non-sudo, and non-shell.
3. Show the exact argv list to the user.
4. Ask for approval unless the user has already approved the exact final action set.

If any check changes, stop and return the command to review.
