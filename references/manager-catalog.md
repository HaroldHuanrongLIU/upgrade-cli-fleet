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

## Trusted Self-Updaters

Use self-updaters only when ownership is clear and the command is known to be a standalone install:

- Bun standalone: `["bun", "upgrade"]`
- Deno standalone: `["deno", "upgrade"]`
- Goose standalone: `["goose", "update"]`
- Cursor Agent standalone: `["cursor-agent", "update"]`
- Claude Code native installer: `["claude", "update"]` when local evidence supports that installer
- uv standalone: `["uv", "self", "update"]` when it is not manager-owned

## Review-Only or Skipped Sources

Keep these in review or skip by default:

- asdf shims
- pnpm standalone installs
- MacPorts paths, because upgrades usually require `sudo`
- Nix profile/store paths
- Conda/Mamba environment paths
- SDKMAN! candidates
- Go binaries under `~/go/bin` or `$GOBIN`
- cargo binaries under `~/.cargo/bin`
- app bundle shims without clear update authority
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

LLM or web enrichment may add category and recipe evidence, but accepted recipes still must pass the safety model.
