# AGENTS.md

Guidance for agents working in this repository.

## What this repo is

A personal library of Agent Skills. It ships **no code and no build** — every file here is either a `SKILL.md`, a reference document, or a helper script belonging to one skill. There is nothing to install, compile, or test.

## Structure

```
skills/<skill-name>/SKILL.md          # required
skills/<skill-name>/references/*.md   # optional, loaded on demand
skills/<skill-name>/scripts/*         # optional, executable
skills/<skill-name>/assets/*          # optional, files to copy verbatim
```

- Directory name **must** equal the frontmatter `name`.
- Names are lowercase, hyphen-separated, gerund- or noun-phrase shaped (`scaffolding-nx-monorepo`, `resolving-merge-conflicts`).

## SKILL.md contract

```markdown
---
name: <matches-directory-name>
description: Use when <concrete trigger situation> — <what it does>.
---

# <Title>

<body>
```

Rules:

- `description` is the **only** thing an agent sees before deciding to load the skill. Write it as a trigger, not a summary. Say when to reach for it and what it covers, in the third person.
- Keep `SKILL.md` under ~500 lines. Push detail into `references/` and link to it by relative path so it loads only when needed.
- Write instructions in the imperative, addressed to the agent that will execute them.
- Prefer a verifiable contract (a table of properties and how to check each) over prose describing an ideal outcome.
- State the failure modes explicitly — what *not* to do, and why the obvious shortcut is wrong. That is usually the highest-value part of a skill.
- Reference files are addressed relative to the skill directory: `references/lint-rules.md`.

## Editing an existing skill

- Read the whole `SKILL.md` before changing part of it; these files are written as a single argument, not a list of tips.
- Keep the voice and density of the file you are editing.
- If a change adds more than ~50 lines of detail, put it in `references/` instead of inlining it.

## Installability constraint

This repo must stay installable with:

```sh
npx skills@latest add dunice/skills
```

That CLI (`vercel-labs/skills`) clones the repo and walks subdirectories for `SKILL.md` files. Two things break it:

- **Never add a `SKILL.md` at the repo root.** The CLI stops descending once it finds one, so a root skill file hides every skill under `skills/` unless the user passes `--full-depth`.
- **Directory name must equal the frontmatter `name`.** That name is what `--skill <name>` matches on.

Verify after any structural change — this lists what the CLI can see without installing:

```sh
npx skills@latest add dunice/skills --list
```

Note for anyone documenting the CLI: `--skill` takes one name and must be **repeated** for several. A comma-separated list is not split and fails with `No matching skills found`.

## Adding a skill

1. `mkdir -p skills/<name>`
2. Write `SKILL.md` with the frontmatter above.
3. Add a row to the **Available skills** table in `README.md`.
4. Confirm the CLI sees it: `npx skills@latest add dunice/skills --list`.

## Conventions

- Markdown only; no linter, formatter, or CI to satisfy.
- `CLAUDE.md` is a symlink to this file — edit `AGENTS.md`, never the symlink.
- Do not add package manifests, lockfiles, or tooling config. This repo stays dependency-free.
