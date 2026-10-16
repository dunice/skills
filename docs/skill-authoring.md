# Skill authoring

## Layout

```
skills/<skill-name>/SKILL.md          # required
skills/<skill-name>/references/*.md   # optional, loaded on demand
skills/<skill-name>/scripts/*         # optional, executable helpers
skills/<skill-name>/assets/*          # optional, files to copy verbatim
```

Name skills in lowercase with hyphens, shaped as a gerund or noun phrase (`scaffolding-nx-monorepo`, `resolving-merge-conflicts`).

## SKILL.md contract

```markdown
---
name: <matches-directory-name>
description: Use when <concrete trigger situation> — <what it does>.
---

# <Title>

<body>
```

- The agent sees only `description` when it decides whether to load the skill. Write it as a trigger: open with "Use when", then say what situation calls for the skill and what it covers.
- Keep `SKILL.md` under ~500 lines. Move detail into `references/` and link to it.
- Link a reference by a bare path relative to the skill directory: `references/lint-rules.md`. Leave out `#section` anchors, because agents open the link as a file path and an anchor breaks that read.
- Prefer a verifiable contract, such as a table of properties and how to check each one, over prose that describes an ideal outcome.
- State the failure modes explicitly: what not to do, and why the obvious shortcut is wrong. This is usually the most valuable part of a skill.

## Editing an existing skill

- Read the whole `SKILL.md` before you change any part of it. Each file is written as one argument, not as a list of separate tips.
- If a change adds more than ~50 lines of detail, put that detail in `references/` instead.
