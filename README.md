# Upgrade CLI Fleet

`upgrade-cli-fleet` is a Codex skill for planning safe upgrades across many local CLI tools at once. It emphasizes owner-aware inventory, package-manager evidence, argv-only executable actions, and explicit user approval before applying changes.

## Contents

- `SKILL.md`: skill entry point, core workflow, and reference routing.
- `references/safety-model.md`: executable action rules and blocked recipe shapes.
- `references/manager-catalog.md`: trusted managers, self-updaters, and review-only sources.
- `references/llm-enrichment.md`: rules for using web or LLM context without overriding local evidence.
- `references/plan-format.md`: structured upgrade plan and audit output format.
- `agents/openai.yaml`: UI metadata for the skill.

## Validation

Run the skill validator after edits:

```sh
python3 /Users/haroldhuanrongliu/.codex/skills/.system/skill-creator/scripts/quick_validate.py .
```
