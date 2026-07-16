# Classification Guide

Use this to bucket any discovered CLI into one of four states before planning an upgrade. Local evidence (path, owner, symlink, PATH order, manager metadata) is authoritative; docs and LLM output are supporting facts.

## The Four Buckets

| Bucket | Meaning | Decision signal |
| --- | --- | --- |
| **A Upgrade** | Safe recipe exists, passes the safety model | trusted manager or bounded native updater, resolved ownership/channel, known target bound or set for pre-apply recheck, low risk, non-sudo, non-shell, argv-native executor |
| **B Skip** | Out of scope — do not manage | app-bundle shim, dependency/non-CLI helper, language runtime, review-only source, macOS system path |
| **C No upgrade needed** | Checked, already current | a non-mutating check reports current, or an approved action is independently verified as a no-op — a runtime outcome, NOT a static roster |
| **D Cannot auto-upgrade** | Safety model blocks automatic execution | real filesystem evidence requires sudo, only a shell/installer path exists, interaction is required, no safe channel-matched target can be established, or no argv-native executor is available |

`review`/`unknown` is the **pre-resolution** state. After investigation the CLI either lands in A, B, C, or D, or remains review because the available evidence is insufficient.

## Default Classification Rules

To keep the skill plug-and-play, apply these defaults without asking the user:

| Situation | Default bucket | Do not ask unless... |
|---|---|---|
| Trusted manager owns it and channel target/binding, low-risk argv, verification, and argv-native execution are available | A | any required evidence or execution capability is missing |
| App-bundle shim, language runtime, dependency helper, or review-only source | B | the user explicitly asks to audit it |
| Non-mutating manager or self-updater check reports "already latest" | C | — |
| Real filesystem evidence requires sudo, or only curl-to-shell/eval/source installer exists | D | — |
| Standalone self-updater with unverified non-interactive flag, or known interactive TUI | D③ / manual-only | the user verifies a working `--yes` or `--non-interactive` flag |
| No safe channel-matched target evidence, or no argv-native execution API | D④ / manual-only | the missing evidence or executor becomes available |
| Recipe is medium/high risk or changes a broad environment | D⑤ / manual-only | the user explicitly requests a separately scoped action |
| Ambiguous ownership, shadowed PATH entry, or conflict | review | — |

## Bucket A — Upgrade

Recipes live in `manager-catalog.md`. Promote a CLI here only when local ownership/channel is resolved, a channel-matched target is pinned or set for immediate pre-apply recheck, the recipe is low-risk, and an argv-native executor is available. A cataloged native self-update subcommand satisfies only the recipe-shape requirement.

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

This is a **runtime outcome**, not a roster. Use a non-mutating check before approval; never run the mutating updater merely to see whether it reports "already latest." After an approved action, independent verification may also establish that the action was a no-op. A CLI current today may be outdated next week, so never publish this bucket as a fixed list.

## Bucket D — Cannot auto-upgrade

The only upgrade path is blocked by the safety model.

**D① — needs sudo.** Real filesystem owner/mode evidence shows the self-updater cannot replace the binary or target in its parent directory without `sudo` (blocked). A sandbox `EPERM` or denied write probe is not enough; report that as an execution-environment limitation instead.

Example: `rclone` at `/usr/local/bin/rclone` (`root:wheel` — root-owned; the exact mode, whether `755` or `111`, is irrelevant once the running user cannot write the path). `rclone selfupdate` exists but cannot write the path without sudo.

**D② — only curl-to-shell / installer / eval / source, no native update.** The tool's docs say "rerun `curl … | sh`" and it has NO `<cmd> update` subcommand. Constructing that argv is blocked.

**D③ — interactive TUI confirmation required and cannot be satisfied.** The bounded `<cmd> update/upgrade` exists, but it renders an interactive menu (e.g., "Install update now" vs "Continue with current version") that requires a real terminal. Agent-run Bash is non-interactive, and the tool has no working `--yes` / `--non-interactive` flag. Classify as manual-only: tell the user to run the bounded subcommand in their own terminal. Do not pipe fake input to the TUI.

**D④ — safe target evidence or execution channel unavailable.** A bounded updater may exist, but no non-mutating check or official channel manifest can establish what this installation will receive, or the available execution tool accepts only shell command text rather than argv tokens. Classify as manual-only. Do not borrow a target version from another install channel and do not serialize argv into a shell string.

**D⑤ — elevated or broad risk.** The only recipe is medium/high risk or changes a broad environment, such as a git pull plus dependency reinstall. Keep it manual-only or request a separately scoped plan; do not mix it into the low-risk fleet batch.

**Critical distinction — a tool's OWN native update that internally curls is ALLOWED.** `codex update`, `kilo upgrade`, and `opencode upgrade` each report "Using method: curl" internally. That is the tool's own implementation (like `brew upgrade` running build scripts), NOT you constructing a curl-to-shell argv. The clean argv `["codex", "update"]` passes the recipe-shape safety check, but it belongs in Bucket A only after target, risk, ownership, and executor checks also pass. Only when YOU would have to write the `curl … | sh` is it D②.

Observed: no D② instance — every CLI that documented curl-to-shell also exposed a native `<cmd> update`.

## Worked examples

| CLI | Bucket | Signal | argv / action | Evidence |
| --- | --- | --- | --- | --- |
| gh | A | brew-owned; exact formula target from guarded brew metadata | `recheck-before-apply`; argv `["/opt/homebrew/bin/brew", "upgrade", "gh"]`; fixed no-auto-update env | `/opt/homebrew` prefix |
| @github/copilot | A | npm-global; exact package/prefix target | `["npm", "install", "-g", "@github/copilot@<target-version>"]` | `npm root -g` |
| bun | D④ / manual-only | bounded updater but no safe channel-matched target evidence in the plan | user runs `["bun", "upgrade"]` | `~/.bun/bin/bun`, not brew |
| codex | A only with channel evidence | standalone marker plus official manifest used by that installer | `recheck-before-apply`; `["codex", "update"]` | `~/.codex/packages/standalone/current` |
| agent | D④ / manual-only | cursor-agent invoked as `agent`, but no safe channel-matched target evidence | user runs `["agent", "update"]` | `~/.local/bin/agent` → cursor-agent version dir |
| hermes | D⑤ / manual-only | editable git + dependency reinstall is medium risk | user runs `["hermes", "update"]` | `~/.hermes/hermes-agent/venv`, editable `.pth` |
| iflow | A | npm global in an exact non-active prefix | `["npm", "install", "-g", "--prefix", "/Users/example/.local", "@iflow-ai/iflow-cli@<target-version>"]` | `/Users/example/.local/bin/iflow` → matching node_modules |
| ollama | B | app-bundle shim | skip (Ollama.app auto-updates) | `/usr/local/bin/ollama` → `Ollama.app` |
| rclone | D① | root-owned, selfupdate needs sudo | blocked | `/usr/local/bin/rclone` `root:wheel` |

## Field notes (operational traps)

1. **npm globals span multiple prefixes.** `npm update -g` only hits the active npm's prefix. Enumerate ALL global prefixes: every nvm node (`~/.nvm/versions/node/*/lib/node_modules`), `~/.npm-global`, `~/.local/lib/node_modules`, and any `--prefix` in `~/.npmrc`. Observed: `iflow` in `~/.local/lib/node_modules` was missed by an nvm-prefixed `npm update -g`. For a non-active prefix, use exact-target `npm install -g --prefix <abs-path> <pkg>@<target-version>`; `update -g` against a foreign prefix is unreliable, while `install -g --prefix` targets the intended prefix. Resolve the prefix to an absolute path in the argv (argv bypasses shell `~` expansion; see `safety-model.md`).
2. **argv[0] name ≠ canonical name.** Resolve the actual PATH name; do not assume the catalog name. A symlink with a different `basename` changes how the tool sees itself (cursor-agent installed as `agent`; the launcher reads `$(basename "$0")`). Re-resolve argv[0] at apply time.
3. **Check ownership and writability before recommending self-update.** `ls -l`/`stat` the resolved binary, replacement target, and parent directory. Only real filesystem evidence can establish D①; keep sandbox denial separate.
4. **"Already latest" is a valid outcome.** Establish it with a non-mutating check before approval or independent verification after an approved update. Do not run `<cmd> update` as a pre-approval probe.
5. **Dangling symlinks are cleanup candidates, not upgrades.** A symlink whose target no longer exists (e.g., a removed uv-tool leaving `~/.local/bin/kimi-legacy`) is stale. Surface for `rm`; do not invent an upgrade.
6. **Surface real PATH-order conflicts/shadowing.** The same command in two PATH dirs with different canonical targets or owners (OrbStack vs `/usr/local`; nvm node vs `~/.local/bin` node) is a conflict and the earlier entry wins. Repeated entries or aliases that resolve to the same target/owner are de-duplicated evidence, not separate conflicts.
7. **Long self-updates may background.** Large-binary updates (e.g., `droid update`) can exceed the foreground threshold and run in the background; poll the task output, then freshly re-resolve and query the installed version before declaring success.
8. **Match the install channel.** A GitHub release, npm dist-tag, Homebrew formula, app release, and standalone updater may expose different versions at the same time. Only the local owner/channel's metadata or the updater's actual official manifest establishes the target.
9. **Preserve lifecycle-script warnings.** If npm reports blocked/skipped install scripts but the version changes, run the approved bounded verification. Report partial/warning when the script could affect the CLI; use Upgraded with a warning note only when it is demonstrably unrelated/optional and all required checks pass. Do not add script approvals or remediation outside the approved action set.
