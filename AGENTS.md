# AGENTS.md

A personal library of Agent Skills, installed with `npx skills@latest add dunice/skills`. It has no build, no lint, no CI and no test suite. The one check is the skills CLI listing, run against the working tree after any change to the directory layout or a frontmatter `name`:

```sh
npx skills@latest add . --list
```

## Guardrails

These hold on every task, because breaking one hides skills from the installer:

- Keep every `SKILL.md` at `skills/<name>/SKILL.md`, with the directory name equal to the frontmatter `name`. A `SKILL.md` at the repo root hides every skill under `skills/`.
- Keep the repo dependency-free: no package manifests, lockfiles, tooling config or `.claude-plugin/` manifests.
- Edit `AGENTS.md`. `CLAUDE.md` is a symlink to it.

## Docs

- Writing or editing a `SKILL.md` or a reference file: [docs/skill-authoring.md](docs/skill-authoring.md)
- Adding a new skill: [docs/adding-a-skill.md](docs/adding-a-skill.md)
- Changing how the repo installs, or documenting the CLI in `README.md`: [docs/installability.md](docs/installability.md)
