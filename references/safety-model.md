# Safety Model

Use ownership resolution before upgrade execution:

```text
PATH inventory -> ownership resolution -> trusted plan -> safety gate -> approval -> apply -> verify
```

## Hard Rules

- Never execute unknown binaries.
- Never probe unknown commands with `--help`, `--version`, update flags, or installer flags.
- Treat local path, canonical path, symlink status, package metadata, and PATH order as objective evidence.
- Treat web pages and LLM output as supporting facts, not proof of local ownership.
- Keep macOS system paths skipped unless the user explicitly asks for audit-only inventory.
- Surface conflicts and shadowed commands for review.
- Prefer owner/package-manager upgrades over tool-specific self-updaters.
- Before approval of the exact final action set, do not run anything that can modify installed software or package-manager installation state.
- Metadata checks may write caches or temporary files only when they cannot change installed software; disclose those writes and disable implicit manager self-update where supported. If that guarantee cannot be made, do not run the check automatically.
- Record subprocess recipes as argv arrays, not shell strings. Never serialize an approved argv back into shell text.
- Execute upgrades only through an argv-native API that preserves token boundaries. A terminal tool whose input is one shell command string is not argv-native; keep the action manual-only unless a trusted argv-native executor is available.
- Block shell-string-only host interfaces by default. A read-only query against a known package manager is an exception only when the host explicitly guarantees no login/profile startup, no aliases/functions, and no expansion, substitution, globbing, redirection, or control operators. The command may then contain only fixed allowlisted environment assignments, the validated absolute manager path, and fixed literal arguments, and must map one-to-one to the recorded argv/environment. Do not interpolate package names, user input, or unknown data. This narrow exception never applies to update/apply commands.
- Block `sudo` by default.
- Block shell wrappers such as `sh -c`, `bash -c`, `zsh -c`, `/usr/bin/env sh`, `/usr/bin/env bash`, and `/usr/bin/env zsh`.
- Block `eval`, `source`, curl-to-shell, and installer scripts.
- Treat `--yes` or equivalent confirmation flags as valid only for trusted low-risk actions.
- Check the resolved binary, replacement target, and relevant parent directory before recommending a self-update. Classify it as needing `sudo` only when real filesystem owner/mode evidence shows the running user cannot replace it.
- Treat sandbox denials, `EPERM` from a restricted agent environment, and denied write probes as execution-environment evidence, not proof of real OS ownership or writability. Do not recommend `sudo` or `chown` from sandbox evidence alone.

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
["pipx", "upgrade", "ruff"]
```

```json
["uv", "tool", "upgrade", "ruff"]
```

A native update subcommand whose internals fetch via curl (e.g., `["codex", "update"]`) is also accepted — see "Tool-Internal Fetch Is Not curl-to-shell" above.

Every token must be literal. argv arrays bypass the shell, so `~`, `$HOME`, and other env refs are NOT expanded at exec time — resolve them to absolute paths (e.g., `/Users/<user>/.local`, not `~/.local`) before forming the argv.

## Apply Rules

Before applying:

1. Re-check that the program named in `argv[0]` resolves to the same trusted path used during planning. The name on PATH may differ from the canonical name (e.g., cursor-agent installed as `agent`). For a bounded `/usr/bin/env` prefix containing only fixed assignments, also re-check the first program after those assignments.
2. Re-run the same non-mutating target query. If the approved target changed, stop and obtain approval for the new target/action.
3. Prefer an argv that pins the approved version when the manager documents safe pinning. For an unpinned latest-style action, disclose the residual check-to-install race.
4. Re-check that the action is still low-risk, non-sudo, and non-shell.
5. Re-check that the executor accepts argv tokens directly. Do not substitute a shell command string.
6. Show the exact argv list and any fixed environment variables to the user.
7. Ask for approval unless the user has already approved that exact target and final action set.

If any check changes, stop, reclassify the affected item, present the revised final action set, and obtain fresh approval before executing any remaining action.

An unexpected privilege request, shell fallback, path change, or ownership change stops the remaining batch. An ordinary package-level failure, warning, or ambiguous verification does not block unrelated package actions. Stop only retries/dependents of that package, or the remaining manager/prefix scope when the manager reports shared database, prefix, or environment corruption.

## Post-Apply Verification

After every approved action:

1. Freshly resolve the command and confirm it still points to the expected owner/channel.
2. Query the installed version through manager metadata or a documented bounded version/check argv.
3. Compare the observed version with the approved channel-matched target.
4. Record lifecycle-script, checksum, health-check, or replacement warnings without adding unapproved remediation.

Report **Upgraded** only when fresh evidence confirms the expected installed result and every required bounded verification passes. Exit code 0 or updater output alone is insufficient. Use **Partial / warning** when the target version is confirmed but a blocked lifecycle step or required health check leaves health uncertain. Use **Unverified** when verification runs without a definitive, parseable result. Use **Error** when execution/verification exits nonzero, the path/owner changes unexpectedly, or the observed version contradicts the approved target. A non-health-affecting warning may be attached to an otherwise verified **Upgraded** result.
