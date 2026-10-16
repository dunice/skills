# Installability

The only supported install path is the `skills` CLI (`vercel-labs/skills`). `README.md` documents that path and no other.

The CLI clones the repo and walks its subdirectories looking for `SKILL.md` files:

- It stops descending as soon as it finds a `SKILL.md`. One at the repo root hides everything under `skills/` unless the user passes `--full-depth`.
- `--skill <name>` matches on the frontmatter `name`, so the directory name has to equal it.

To check what the CLI sees in the working tree, without installing anything:

```sh
npx skills@latest add . --list
```

Once changes are pushed, `npx skills@latest add dunice/skills --list` shows what users will get.
