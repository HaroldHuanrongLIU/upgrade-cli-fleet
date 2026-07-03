# Manager Catalog

Use this catalog to resolve ownership and decide upgrade scope. Prefer manager-level upgrades when a command is owned by a trusted package manager.

## Trusted Manager Upgrades

These managers can produce trusted low-risk upgrade actions when local ownership evidence is clear:

| Manager | Typical evidence | Preferred upgrade scope |
| --- | --- | --- |
| Homebrew | path under Homebrew prefix, `brew list`, `brew info` | `["brew", "upgrade"]` or package-specific `brew upgrade <formula>` when requested |
| npm global | global npm prefix/bin, `npm root -g`, `npm list -g --depth=0` | package-specific `["npm", "update", "-g", "<package>"]` or fleet-level npm global update when explicitly requested |
| pipx | pipx venv/bin path, `pipx list --json` | `["pipx", "upgrade", "<package>"]` or `["pipx", "upgrade-all"]` |
| uv tool | uv tool dir metadata, `uv tool list` | `["uv", "tool", "upgrade", "<tool>"]` |
| mise | mise shims/installs, `mise list` | use mise-managed tool upgrade only when the target and scope are clear |
| rustup | rustup toolchains/components | `["rustup", "update"]` |

**npm global prefix caveat:** `npm update -g` only upgrades the *active* npm's prefix. Enumerate every global prefix on the machine — each nvm node (`~/.nvm/versions/node/*/lib/node_modules`), `~/.npm-global`, `~/.local/lib/node_modules`, and any `--prefix` set in `~/.npmrc`. Globals sitting in a non-active prefix are easily missed.

## Trusted Self-Updaters

Use self-updaters only when ownership is clear and the command is known to be a standalone install:

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

- **A tool's own native update that fetches internally is allowed.** `codex`, `kilo`, and `opencode` print "Using method: curl" — that is the tool downloading its own update, equivalent to `brew upgrade` running build scripts. The argv `["codex", "update"]` passes the safety model. You must NOT construct the `curl … | sh` argv yourself; see `safety-model.md`.
- **Resolve the actual argv[0].** A self-updater may be installed under a different name than the canonical one (e.g., cursor-agent installed as `agent`; the launcher reads `$(basename "$0")`). Use the name found on PATH, not the catalog's canonical name.
- **Check the binary is writable.** If the resolved path is root-owned or not user-writable, the self-update needs `sudo` and is blocked — classify as cannot-upgrade (Bucket D①), not a trusted self-updater.

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

LLM or web enrichment may add category and recipe evidence, but accepted recipes still must pass the safety model. When a bounded `<cmd> update/upgrade` exists and ownership resolves, promote from review to a trusted self-updater (above).
