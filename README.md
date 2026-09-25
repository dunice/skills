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
| [`writing-typescript`](skills/writing-typescript/SKILL.md) | Writing or reviewing TypeScript — external input validated at one boundary, type guards instead of `as`, new values instead of mutation, tagged errors instead of strings. |
| [`refactoring-safely`](skills/refactoring-safely/SKILL.md) | Restructuring code that already works — pin the behavior in tests first, one named move per commit, shape and behavior never in the same commit. |
| [`writing-commit-messages`](skills/writing-commit-messages/SKILL.md) | About to run `git commit`, amend or reword, or splitting a day of mixed work — one-line Conventional Commits subject by default, no attribution trailers. |
| [`managing-github-pull-requests`](skills/managing-github-pull-requests/SKILL.md) | Opening or updating a PR with `gh` — draft by default, body from a file, reviewers, labels, screenshots, ready-for-review. |
| [`automating-browsers-with-agent-browser`](skills/automating-browsers-with-agent-browser/SKILL.md) | Driving a real browser with the `agent-browser` CLI — load its version-matched guide first, one named session per task, re-snapshot after every page change. |
| [`writing-pull-request-descriptions`](skills/writing-pull-request-descriptions/SKILL.md) | Opening a PR or rewriting its description — self-review and scope gates first, then a four-section body a reviewer can predict the diff from. |

## Installing

Install every skill in this repo with the [`skills`](https://github.com/vercel-labs/skills) CLI:

```sh
npx skills@latest add dunice/skills
```

Installs project-level into `./.agents/skills/`, symlinked into every agent directory it detects (Claude Code, Codex, Cursor, GitHub Copilot, Warp, and others), and records `skills-lock.json`.

### Installing only some skills

Pass `--skill` (`-s`) with a skill name. Repeat the flag for several skills — a comma-separated list is **not** parsed and fails with `No matching skills found`:

```sh
npx skills@latest add dunice/skills --skill scaffolding-nx-monorepo

# several skills: repeat the flag
npx skills@latest add dunice/skills --skill skill-one --skill skill-two
```

List what a repo offers first, without installing anything:

```sh
npx skills@latest add dunice/skills --list
```

Other flags worth knowing:

| Flag | Effect |
|------|--------|
| `-s, --skill <name>` | Install only that skill (`'*'` for all). Repeat for several. |
| `-a, --agent <agent>` | Install for that agent only, e.g. `--agent claude-code`, instead of every agent detected. |
| `-g, --global` | Install user-level (`~/.claude/skills/`) instead of project-level. |
| `-l, --list` | Show available skills and exit. |
| `-y, --yes` | Skip prompts. Implied when a coding agent is driving. |
| `--all` | Shorthand for `--skill '*' --agent '*' -y`. |

Update later with `npx skills@latest update`, remove with `npx skills@latest remove -s scaffolding-nx-monorepo`.

### Without the CLI

Symlink directly into a directory Claude Code scans:

```sh
git clone git@github.com:dunice/skills.git ~/Projects/dunice/skills
ln -s ~/Projects/dunice/skills/skills/scaffolding-nx-monorepo \
      ~/.claude/skills/scaffolding-nx-monorepo
```

Project-scoped instead: link into `<project>/.claude/skills/`.

Claude reads only the frontmatter `description` until a task matches, then loads the body. Nothing is loaded eagerly, so a large collection costs almost no context.

## Adding a skill

See [AGENTS.md](AGENTS.md) for the writing conventions this repo follows.
