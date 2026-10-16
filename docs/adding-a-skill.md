# Adding a skill

1. `mkdir -p skills/<name>`
2. Write `SKILL.md` to the contract in [skill-authoring.md](skill-authoring.md).
3. Add a row to the **Available skills** table in `README.md`.
4. Run `npx skills@latest add . --list`. The step is done when `<name>` appears in the output with its description.
