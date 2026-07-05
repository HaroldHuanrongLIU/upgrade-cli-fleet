# Classification Guide

Use this to bucket any discovered CLI into one of four states before planning an upgrade. Local evidence (path, owner, symlink, PATH order, manager metadata) is authoritative; docs and LLM output are supporting facts.

## The Four Buckets

| Bucket | Meaning | Decision signal |
| --- | --- | --- |
| **A Upgrade** | Safe recipe exists, passes the safety model | trusted manager owns it, OR a documented bounded `<cmd> update/upgrade` self-updater with resolved ownership, non-sudo, non-shell |
| **B Skip** | Out of scope — do not manage | app-bundle shim, dependency/non-CLI helper, language runtime, review-only source, macOS system path |
| **C No upgrade needed** | Checked, already current | the upgrade/check command ran and reported "already latest" — a runtime outcome, NOT a static roster |
| **D Cannot upgrade** | Safety model blocks the only path | binary at root-owned/non-user-writable path (self-update needs sudo), OR the only update method is curl-to-shell/eval/source/installer with NO native `<cmd> update` |

`review`/`unknown` is the **pre-resolution** state. After investigation the CLI lands in A, B, C, or D.

## Bucket A — Upgrade

Recipes live in `manager-catalog.md`. Promote a CLI here only when a trusted manager owns it (Homebrew/npm-global/pipx/uv-tool/mise/rustup) or it has a bounded self-update subcommand AND local ownership is resolved.

## Bucket B — Skip (out of scope)

These are updated by their app/runtime/manager or are not CLI upgrade targets. Do not manage them via the fleet.

| Signal | Examples |
| --- | --- |
| App-bundle shim (binary symlinks into `*.app/Contents/MacOS`) | ollama→Ollama.app, kaku/k→Kaku.app, kiro-cli-chat/term→Kiro CLI.app, OrbStack docker/kubectl/orb/orbctl, Tailscale, Obsidian |
| Dependency or non-CLI helper | f2py, numpy-config, uvx, corepack, the bundled shells (bash/fish/nu/zsh shipped with kiro-cli-term) |
| Language runtime / version-manager target | nvm's node, conda envs, mise/asdf-managed tools |
| Review-only source by policy | conda/MacPorts/Nix/SDKMAN, `~/go/bin`, `~/.cargo/bin` (rustup owns these), pnpm standalone |
| macOS system path | `/usr/bin`, `/bin`, `/Library/Apple/usr/bin`, `/System` |

## Bucket C — No upgrade needed

This is a **runtime outcome**, not a roster. Always run the check; "already latest" is a valid no-op. A CLI current today may be outdated next week, so never publish this bucket as a fixed list.

## Bucket D — Cannot upgrade

The only upgrade path is blocked by the safety model.

**D① — needs sudo.** The binary sits at a root-owned or non-user-writable path, so its self-update cannot replace it without `sudo` (blocked). Detect with `ls -l`/`stat` on the resolved binary path: owner `root` or mode lacks user-write → blocked.

Example: `rclone` at `/usr/local/bin/rclone` (`root:wheel` — root-owned; the exact mode, whether `755` or `111`, is irrelevant once the running user cannot write the path). `rclone selfupdate` exists but cannot write the path without sudo.

**D② — only curl-to-shell / installer / eval / source, no native update.** The tool's docs say "rerun `curl … | sh`" and it has NO `<cmd> update` subcommand. Constructing that argv is blocked.

**D③ — interactive TUI confirmation required and cannot be satisfied.** The bounded `<cmd> update/upgrade` exists, but it renders an interactive menu (e.g., "Install update now" vs "Continue with current version") that requires a real terminal. Agent-run Bash is non-interactive, and the tool has no working `--yes` / `--non-interactive` flag. Classify as manual-only: tell the user to run the bounded subcommand in their own terminal. Do not pipe fake input to the TUI.

**Critical distinction — a tool's OWN native update that internally curls is ALLOWED.** `codex update`, `kilo upgrade`, and `opencode upgrade` each report "Using method: curl" internally. That is the tool's own implementation (like `brew upgrade` running build scripts), NOT you constructing a curl-to-shell argv. The clean argv `["codex", "update"]` passes the safety model and belongs in Bucket A. Only when YOU would have to write the `curl … | sh` is it D②.

Observed: no D② instance — every CLI that documented curl-to-shell also exposed a native `<cmd> update`.

## Worked examples (medium)

| CLI | Bucket | Signal | argv / action | Evidence |
| --- | --- | --- | --- | --- |
| gh | A | brew-owned | `["brew", "upgrade"]` | `/opt/homebrew` prefix |
| @github/copilot | A | npm-global (nvm prefix) | `["npm", "update", "-g"]` | `npm root -g` |
| bun | A→C | standalone self-updater | `["bun", "upgrade"]` (already latest) | `~/.bun/bin/bun`, not brew |
| codex | A | standalone, native update fetches internally | `["codex", "update"]` | `~/.codex/packages/standalone/current` |
| agent | A | cursor-agent invoked as `agent` | `["agent", "update"]` | `~/.local/bin/agent` → cursor-agent version dir |
| hermes | A | venv editable (git + uv), medium risk | `["hermes", "update"]` | `~/.hermes/hermes-agent/venv`, editable `.pth` |
| iflow | A | npm global in `~/.local` prefix | `npm i -g --prefix ~/.local @iflow-ai/iflow-cli` | `~/.local/bin/iflow` → `~/.local/lib/node_modules` |
| ollama | B | app-bundle shim | skip (Ollama.app auto-updates) | `/usr/local/bin/ollama` → `Ollama.app` |
| rclone | D① | root-owned, selfupdate needs sudo | blocked | `/usr/local/bin/rclone` `root:wheel` |

## Field notes (operational traps)

1. **npm globals span multiple prefixes.** `npm update -g` only hits the active npm's prefix. Enumerate ALL global prefixes: every nvm node (`~/.nvm/versions/node/*/lib/node_modules`), `~/.npm-global`, `~/.local/lib/node_modules`, and any `--prefix` in `~/.npmrc`. Observed: `iflow` in `~/.local/lib/node_modules` was missed by an nvm-prefixed `npm update -g`. For a non-active prefix, prefer `npm i -g --prefix <abs-path> <pkg>` over `npm update -g --prefix <pkg>` — `update -g` against a foreign prefix is unreliable, while `install -g --prefix` reliably pulls latest into that prefix. Resolve the prefix to an absolute path in the argv (argv bypasses shell `~` expansion; see `safety-model.md`).
2. **argv[0] name ≠ canonical name.** Resolve the actual PATH name; do not assume the catalog name. A symlink with a different `basename` changes how the tool sees itself (cursor-agent installed as `agent`; the launcher reads `$(basename "$0")`). Re-resolve argv[0] at apply time.
3. **Check ownership and writability before recommending self-update.** `ls -l`/`stat` the resolved binary. Root-owned or non-user-writable → self-update needs sudo → Bucket D①, not A.
4. **"Already latest" is a valid outcome.** Running `<cmd> update` and getting "up to date" is success, not failure. Report it; do not hunt for a different command.
5. **Dangling symlinks are cleanup candidates, not upgrades.** A symlink whose target no longer exists (e.g., a removed uv-tool leaving `~/.local/bin/kimi-legacy`) is stale. Surface for `rm`; do not invent an upgrade.
6. **Surface PATH-order conflicts/shadowing.** The same command in two PATH dirs (OrbStack vs `/usr/local`; nvm node vs `~/.local/bin` node) — the earlier entry wins. Record both and flag which is active.
7. **Long self-updates may background.** Large-binary updates (e.g., `droid update`) can exceed the foreground threshold and run in the background; poll the task output before declaring done.
