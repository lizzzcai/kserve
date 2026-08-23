# KServe Agent Skills — Authoring Guidelines

## Skill Directory Layout

```
skills/
├── <skill-name>/
│   ├── SKILL.md          # Required: frontmatter + decision logic only
│   └── references/       # Optional: detail files loaded on demand
```

## Design Principles

- **Progressive disclosure**: keep `SKILL.md` under ~120 lines of decision logic. Move detail to `references/` files. The progressive disclosure table must name the exact trigger that causes the agent to load each file — not just "see references/ for details."
- **Descriptions as routing contracts**: frontmatter `description` must include explicit WHEN triggers and `Don't use for` exclusions so agents activate the right skill and hand off correctly.
- **Non-interactive context discovery first**: use `kubectl`, `git`, or env inspection to resolve name/namespace/context before asking the user. Ask only for what cannot be inferred.
- **Decision trees over prose**: use tables with concrete branch conditions, not narrative paragraphs. Each branch should resolve to a specific next step.
- **Read-only boundary before writes**: diagnostic skills must establish root cause before proposing any mutation. State this explicitly as a Rule.
- **Numbered rules for hard constraints**: list mandatory constraints at the top as short imperative rules — not guidelines. Cover the most common silent-failure modes.
- **Gotchas inline, not in references**: non-obvious traps that cause silent failures stay in `SKILL.md` body so the agent reads them before encountering the situation.
- **Route to live sources, never embed**: `SKILL.md` contains no website URLs or version numbers. Those belong in `references/` files pointing to `https://kserve.github.io/website/docs/...` and `kserve-deps.env`. Skills stay correct as docs evolve without touching `SKILL.md`.
- **Cross-validate all factual claims**: every field name, condition type, label, and runtime name must be verified against source code before committing. Never invent reason strings or label values.

## References

- [agentskills.io specification](https://agentskills.io/specification)
- [agentskills.io best practices](https://agentskills.io/skill-creation/best-practices)
