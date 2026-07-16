# Manager Catalog

Use this catalog to resolve ownership and decide upgrade scope. Prefer package-specific manager upgrades when a command is owned by a trusted package manager.

## Trusted Manager Upgrades

These managers can produce trusted low-risk upgrade actions when local ownership evidence is clear:

| Manager | Typical evidence | Preferred upgrade scope |
| --- | --- | --- |
| Homebrew | path under Homebrew prefix, `brew list --formula`, `brew list --cask`, `brew info` | For formulae: `["brew", "upgrade", "<formula>"]`. For casks: `["brew", "upgrade", "--cask", "<cask>"]` or `["brew", "install", "--cask", "<cask>"]` when upgrading an installed cask. Determine the install type from `brew list --formula` / `brew list --cask`; do not assume formula. |
| npm global | global npm prefix/bin, `npm root -g`, `npm list -g --depth=0` | prefer exact-target `["npm", "install", "-g", "<package>@<target-version>"]`; do not auto-apply an unpinned fleet-level global update |
| pipx | pipx venv/bin path, `pipx list --json` | `["pipx", "upgrade", "<package>"]` or `["pipx", "upgrade-all"]` |
| uv tool | uv tool dir metadata, `uv tool list` | `["uv", "tool", "upgrade", "<tool>"]` |
| mise | mise shims/installs, `mise list` | use mise-managed tool upgrade only when the target and scope are clear |
| rustup | rustup toolchains/components | skip toolchain/runtime upgrades by default; use `["rustup", "update"]` only when explicitly requested |

**Homebrew cask vs formula caveat:** A command may be installed as a cask (`CodexBar.app`) rather than a formula. Before forming the argv, check `brew list --formula` and `brew list --cask`. If the package appears in the cask list, use `--cask` in the upgrade argv. For example, `codexbar` installed as a cask must be upgraded with `["brew", "upgrade", "--cask", "codexbar"]` or `["brew", "install", "--cask", "codexbar"]`; `["brew", "upgrade", "codexbar"]` will fail with an untrusted-tap or formula-not-found error.

**Homebrew metadata must not auto-update Homebrew:** Do not run plain `brew outdated` before approval. Resolve the absolute `brew` path and disable implicit auto-update through the executor's environment map:

```json
{
  "argv": ["/opt/homebrew/bin/brew", "outdated", "--json=v2"],
  "env": {"HOMEBREW_NO_AUTO_UPDATE": "1"}
}
```

If the executor has no environment map but does accept argv natively, the equivalent bounded form is `["/usr/bin/env", "HOMEBREW_NO_AUTO_UPDATE=1", "/opt/homebrew/bin/brew", "outdated", "--json=v2"]`. Use the resolved Homebrew path, not the example path. Apply the same guard to package upgrades unless a Homebrew self-update is separately listed and approved. Cache writes must be disclosed as diagnostic side effects; changing the installed Homebrew version is not a metadata side effect.

**npm global prefix caveat:** `npm update -g` only upgrades the *active* npm's prefix. Enumerate every global prefix on the machine — each nvm node (`~/.nvm/versions/node/*/lib/node_modules`), `~/.npm-global`, `~/.local/lib/node_modules`, and any `--prefix` set in `~/.npmrc`. Globals sitting in a non-active prefix are easily missed.

**Manager/runtime scope caveat:** Updating packages does not implicitly authorize updating the manager or language runtime. Homebrew itself, npm itself, Node, rustup toolchains, Python runtimes, and similar infrastructure stay skipped unless the user explicitly includes them. Present manager/runtime changes as separate actions with their own targets and evidence.

## Target Version Evidence

The target must come from the local installation's channel, in this order:

1. owning manager metadata for the exact package and prefix;
2. a documented non-mutating check/dry-run from the installed standalone updater;
3. the official release manifest or registry that the installed standalone updater actually consumes.

A GitHub release, npm dist-tag, Homebrew version, or app release from a different channel may explain that newer code exists, but it does not establish the approved target. If the installed updater has no safe check and its exact release channel cannot be tied to an official manifest, keep it manual-only.

Bind the approved version into the argv when the manager documents safe version pinning. Otherwise label the action `recheck-before-apply`, repeat the same target query immediately before execution, and disclose that an unpinned latest-style action retains a small check-to-install race.

## Trusted Self-Updaters

Use self-updaters only when ownership is clear and the command is known to be a standalone install. Listing a recipe here does not by itself make it auto-applicable: a trusted plan also needs safe channel-matched target evidence and post-apply verification.

- Bun standalone: `["bun", "upgrade"]`
- Deno standalone: `["deno", "upgrade"]`
- Goose standalone: `["goose", "update"]`
- Cursor Agent standalone: `["cursor-agent", "update"]`
- Claude Code native installer: `["claude", "update"]` when local evidence supports that installer
- uv standalone: `["uv", "self", "update"]` when it is not manager-owned
- Codex CLI standalone: `["codex", "update"]`
- Kimi Code CLI: `["kimi", "upgrade"]`
- Factory Droid: `["droid", "update"]`
- Qoder CLI: `["qodercli", "update"]`
- Kiro CLI: `["kiro-cli", "update"]`
- OpenCode: `["opencode", "upgrade"]`
- Antigravity CLI: `["agy", "update"]`
- Kilo Code CLI: `["kilo", "upgrade"]`
- Hermes Agent (NousResearch): `["hermes", "update"]` — git editable install in a uv-created venv; update does git pull + dependency reinstall. Mark risk `medium`.

### Self-updater notes

- **A tool's own native update that fetches internally is an allowed recipe shape.** `codex`, `kilo`, and `opencode` print "Using method: curl" — that is the tool downloading its own update, equivalent to `brew upgrade` running build scripts. The argv `["codex", "update"]` is not shell-based, but target, ownership, risk, executor, and verification checks must still pass. You must NOT construct the `curl … | sh` argv yourself; see `safety-model.md`.
- **Resolve the actual argv[0].** A self-updater may be installed under a different name than the canonical one (e.g., cursor-agent installed as `agent`; the launcher reads `$(basename "$0")`). Use the name found on PATH, not the catalog's canonical name.
- **Check the binary is writable.** If real filesystem owner/mode evidence shows the resolved target and its parent cannot be replaced by the user, the self-update needs `sudo` and is blocked — classify as cannot-auto-upgrade (Bucket D①), not a trusted self-updater. Do not infer this from a sandbox denial.
- **Separate update from check.** Do not run the mutating update subcommand to discover whether an update exists. Record a distinct documented check/dry-run argv or official channel manifest. If none exists, make the updater manual-only.
- **Check for interactive TUI confirmation.** Some self-updaters (e.g., `kimi upgrade`) render an interactive menu that requires the user to select "Install update" and press Enter. Agent-run Bash is non-interactive and cannot confirm these TUIs. If the tool has no `--yes` / `--non-interactive` equivalent that works in headless mode, classify it as **manual-only** and tell the user to run the bounded subcommand in their own terminal. Do not invent input-piping tricks to fake the TUI.
- **Verify independently afterward.** Re-resolve the installed command and query its version after the updater exits. Download/checksum output and exit code 0 are not sufficient evidence.
- **Preserve npm lifecycle warnings.** If an npm global update reports blocked or skipped install scripts, do not silently add `allow-scripts`, `approve-builds`, or another remediation. Verify the package version and the pre-approved bounded version/smoke check. Report partial/warning whenever a skipped script could affect the installed CLI; report Upgraded with the warning attached only when the skipped script is demonstrably unrelated/optional and every required check passes.

## Review-Only or Skipped Sources

Keep these in review or skip by default:

- asdf shims
- pnpm standalone installs
- MacPorts paths, because upgrades usually require `sudo`
- Nix profile/store paths
- Conda/Mamba environment paths
- SDKMAN! candidates
- Go binaries under `~/go/bin` or `$GOBIN`
- cargo binaries under `~/.cargo/bin` (rustup manages the toolchain)
- app bundle shims without clear update authority (ollama, kaku, OrbStack tools, Tailscale, Obsidian — see `classification-guide.md` Bucket B)
- macOS system paths

## AI Coding CLIs

When installer ownership is unclear, keep known AI coding CLIs as review-only unless local evidence or approved enrichment resolves ownership:

- `codex`
- `claude`
- `gemini`
- `amp`
- `aider`
- `aider-chat`
- `goose`
- `opencode`
- `qwen`
- `cursor-agent`
- `crush`
- `plandex`
- `kiro`
- `kiro-cli`

LLM or web enrichment may add category and recipe evidence, but accepted recipes still must pass the safety model. A bounded `<cmd> update/upgrade` plus resolved ownership identifies a candidate self-updater; promote it to Bucket A only when channel-matched target, low risk, argv-native execution, and post-apply verification are also available.
