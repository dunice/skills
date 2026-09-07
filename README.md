# skills

Personal collection of [Agent Skills](https://code.claude.com/docs/en/skills) — reusable instruction sets that Claude Code (and other agents that read `SKILL.md` files) can load on demand.

## Layout

```
skills/
  <skill-name>/
    SKILL.md          # required: frontmatter + instructions
    references/       # optional: deep-dive docs loaded only when needed
    scripts/          # optional: executable helpers
    assets/           # optional: templates, config files to copy
```

One directory per skill. The directory name is the skill name.

## Available skills

| Skill | Use when |
|-------|----------|
| [`scaffolding-nx-monorepo`](skills/scaffolding-nx-monorepo/SKILL.md) | Starting a new full-stack TypeScript repo — Nx, NestJS, Drizzle + Postgres, React, tRPC, Vitest, Biome, pnpm, Docker, CI. |

## Using these skills

Symlink or clone into a directory Claude Code scans:

```sh
# personal, available in every project
ln -s "$PWD/skills/scaffolding-nx-monorepo" ~/.claude/skills/scaffolding-nx-monorepo

# or link the whole repo
git clone git@github.com:dunice/skills.git ~/Projects/dunice/skills
for d in ~/Projects/dunice/skills/skills/*/; do
  ln -s "$d" ~/.claude/skills/"$(basename "$d")"
done
```

Project-scoped instead: link into `<project>/.claude/skills/`.

Claude reads only the frontmatter `description` until a task matches, then loads the body. Nothing is loaded eagerly, so a large collection costs almost no context.

## Adding a skill

See [AGENTS.md](AGENTS.md) for the writing conventions this repo follows.
