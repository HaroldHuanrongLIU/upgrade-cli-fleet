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
- Check the resolved binary's owner and writability before recommending a self-update: if the path is root-owned or not user-writable, the self-update needs `sudo` and is blocked.

## Tool-Internal Fetch Is Not curl-to-shell

The curl-to-shell block forbids YOU from constructing `curl … | sh` argv. It does NOT forbid running a tool's OWN native update subcommand that fetches internally. `codex update`, `kilo upgrade`, and `opencode upgrade` each download their update via curl (they print "Using method: curl") — this is the tool's own implementation, equivalent to `brew upgrade` running build scripts. The argv `["codex", "update"]` is an accepted recipe shape. Only when you would have to write the `curl … | sh` yourself is it blocked.

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

A native update subcommand whose internals fetch via curl (e.g., `["codex", "update"]`) is also accepted — see "Tool-Internal Fetch Is Not curl-to-shell" above.

Every token must be literal. argv arrays bypass the shell, so `~`, `$HOME`, and other env refs are NOT expanded at exec time — resolve them to absolute paths (e.g., `/Users/<user>/.local`, not `~/.local`) before forming the argv.

## Apply Rules

Before applying:

1. Re-check that the program named in `argv[0]` resolves to the same trusted path used during planning. The name on PATH may differ from the canonical name (e.g., cursor-agent installed as `agent`).
2. Re-check that the action is still low-risk, non-sudo, and non-shell.
3. Show the exact argv list to the user.
4. Ask for approval unless the user has already approved the exact final action set.

If any check changes, stop and return the command to review.
