# Repository Instructions

## Repository Purpose

This repository is the `upgrade-cli-fleet` Codex skill. It guides agents through safe, owner-aware planning for upgrading many local CLI tools at once.

## Project Structure

- `SKILL.md`: required skill entry point and routing instructions.
- `references/`: detailed material loaded only when needed.
- `agents/openai.yaml`: UI metadata for the skill.
- `LICENSE`: repository license.

Do not add a Rust, Python, or shell application unless the user explicitly asks for a bundled script. Keep the skill focused on reusable instructions and reference material.

## Editing Guidelines

- Keep `SKILL.md` concise and under 500 lines.
- Frontmatter must contain only `name` and `description`.
- Use lowercase hyphen-case for skill names.
- Put detailed policy, catalogs, schemas, or examples in `references/`.
- Do not add README, changelog, installation guide, or other auxiliary docs unless explicitly requested.
- Preserve the safety model: unknown commands are not executed or probed, `sudo` and shell recipes are blocked by default, and executable actions are argv arrays.

## Validation

Run the skill validator after edits:

```sh
python3 /Users/haroldhuanrongliu/.codex/skills/.system/skill-creator/scripts/quick_validate.py .
```

Use `rg --files` to inspect the skill contents before finishing.
