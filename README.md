# Claude Code Skills Marketplace

Personal marketplace repository for Claude Code skills.

## Repository Layout

```text
.claude-plugin/marketplace.json
plugins/personal-skills/.claude-plugin/plugin.json
plugins/personal-skills/skills/refactor/SKILL.md
plugins/personal-skills/skills/write-intent-spec/SKILL.md
plugins/personal-skills/skills/write-execution-spec/SKILL.md
plugins/personal-skills/skills/review-execution-spec/SKILL.md
plugins/personal-skills/skills/implement-from-spec/SKILL.md
plugins/personal-skills/skills/review-code-spec/SKILL.md
```

## Installation

```bash
/plugin marketplace add <repo-url>
/plugin install skills@personal-skills
```

## Available Skills

- `refactor`: autonomous test-first refactoring for Go and Java codebases.
- `write-intent-spec`: 在代码落地前先锁定需求、约束和风险，输出可审计的 Intent Spec。
- `write-execution-spec`: 基于评审通过的 Intent Spec 生成可执行、可验证、可上线的 Execution Spec。
- `review-execution-spec`: 以守门人视角防御性审查 Execution Spec，按严重级别输出问题与修复验证建议。
- `implement-from-spec`: 严格按 Execution Spec 与 Review Feedback 以 TDD 实现功能，并输出可追溯验证证据。
- `review-code-spec`: 以 Spec-Driven 守门人方式审查代码与 Spec 一致性，拦截 scope creep 与生产风险。

## Add a Skill

1. Create `plugins/personal-skills/skills/<skill-name>/SKILL.md`.
2. Add `"./skills/<skill-name>"` to the `skills` array of `personal-skills` in `.claude-plugin/marketplace.json`.
