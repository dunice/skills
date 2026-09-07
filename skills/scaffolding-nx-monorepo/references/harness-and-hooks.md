# The agent harness

One source of truth — `.agents/` — and every AI tool links into it. Nothing is duplicated, so a skill edited once is edited for all tools.

## Layout

```
.agents/skills/      <name>/SKILL.md      — reusable procedures
.agents/workflows/   <name>.md            — slash commands
.agents/agents/      <name>.md            — subagents
.agents/rules/       NN-<area>.mdc        — always-on scoped rules
```

`workflows/`, not `commands/` — that is the directory name `harness-score` recognises under `.agents/`. Claude Code reads it as commands through a symlink.

## Symlinks

Claude Code does not read `.agents/`, so `.claude/` links into it. Cursor reads `.agents/skills/` natively, so `.cursor/` must hold **no subdirectories** — a `.cursor/skills/` would be detected twice.

```
.claude/agents    -> ../.agents/agents
.claude/commands  -> ../.agents/workflows
.claude/skills    -> ../.agents/skills
.claude/settings.json    (real file — Claude-specific hook config)
.cursor/hooks.json       (real file — Cursor-specific hook config)
CLAUDE.md         -> AGENTS.md
```

Create them with relative targets so the repo stays portable:

```bash
ln -s ../.agents/skills    .claude/skills
ln -s ../.agents/workflows .claude/commands
ln -s ../.agents/agents    .claude/agents
ln -s AGENTS.md            CLAUDE.md
```

Git stores symlinks natively; commit them as-is.

## Front matter

| File | Required front matter |
|---|---|
| `skills/<n>/SKILL.md` | `name`, `description` (starts "Use when…") |
| `workflows/<n>.md` | `description` |
| `agents/<n>.md` | `name`, `description`, `tools` |
| `rules/NN-<area>.mdc` | `description`, `alwaysApply: true` (or `globs:` to scope) |

## Hooks — one script, two harnesses

Both tools point at the same file under `scripts/`. Neither config contains inline shell; the config only names a script.

**`.claude/settings.json`**

```jsonc
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "hooks": {
    "PreToolUse":  [{ "matcher": "Bash",
      "hooks": [{ "type": "command", "command": "node \"${CLAUDE_PROJECT_DIR}/scripts/guard-shell.mjs\"", "timeout": 10 }] }],
    "PostToolUse": [{ "matcher": "Write|Edit|MultiEdit|NotebookEdit",
      "hooks": [{ "type": "command", "command": "node \"${CLAUDE_PROJECT_DIR}/scripts/format-on-edit.mjs\"", "timeout": 60 }] }]
  }
}
```

**`.cursor/hooks.json`**

```json
{
  "version": 1,
  "hooks": {
    "beforeShellExecution": [{ "command": "node ./scripts/guard-shell.mjs" }],
    "afterFileEdit":        [{ "command": "node ./scripts/format-on-edit.mjs" }]
  }
}
```

### The protocol differences

The decision in the middle is identical; only the payload shape and the reply differ. Keep both in one `scripts/lib/harness.mjs`:

| | Claude Code | Cursor |
|---|---|---|
| Event | `PreToolUse` / `PostToolUse` | `beforeShellExecution` / `afterFileEdit` |
| Payload | fields nested under `tool_input`, plus `hook_event_name` | fields flat |
| Allow | **emit nothing** (falls through to the normal permission flow) | must emit `{"permission":"allow"}` |
| Deny | `{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny","permissionDecisionReason":"…"}}` | `{"permission":"deny","userMessage":"…","agentMessage":"…"}` |
| Surface feedback to the model | exit code **2** with the message on stderr | exit 0 |

Detect from the payload shape:

```js
export function detectHarness(payload) {
  return 'hook_event_name' in payload || 'tool_input' in payload ? 'claude' : 'cursor';
}

export function readField(payload, field) {
  return payload?.tool_input?.[field] ?? payload?.[field];
}
```

### Gate hook — `scripts/guard-shell.mjs`

Allow everything; deny four families of irreversible or externally visible commands. Regexes worth copying, because the naive versions miss real cases:

```js
export const DENY_RULES = [
  { id: 'publish',
    test: /\b(?:npm|pnpm|yarn|bun)\s+publish\b/,
    reason: 'Publishing to a registry is irreversible and externally visible. Releases go through CI.' },

  { id: 'force-push',
    // covers --force, --force-with-lease, -f, and the `+ref:ref` refspec form
    test: /\bgit\s+push\b[^\n]*(?:\s--force(?:-with-lease)?\b|\s-f\b|\s\+[\w./-]+:)/,
    reason: 'Force pushing rewrites published history.' },

  { id: 'reset-hard',
    test: /\bgit\s+reset\b[^\n]*\s--hard\b/,
    reason: 'Destroys uncommitted work with no undo. Use `git stash` or `git restore <path>`.' },

  { id: 'recursive-delete',
    // -r anywhere in a bundled flag, --recursive, rimraf, and `find … -delete`
    test: /(?:\brm\s+(?:-[a-zA-Z]*[rR][a-zA-Z]*|--recursive)\b)|(?:\brimraf\b)|(?:\bfind\b[^\n]*\s-delete\b)/,
    reason: 'Recursive deletion is the classic agent catastrophe.' },
];
```

Every reason names the safe alternative. A gate that only says "no" gets worked around; a gate that says "use `git restore` instead" gets obeyed.

The gate is the answer, not an obstacle — document in `AGENTS.md` that a blocked command must be handed to a human, never re-spelled to slip past the regex.

### Feedback hook — `scripts/format-on-edit.mjs`

Runs `biome check --write --no-errors-on-unmatched <file>` on the file just written, then hands back whatever Biome could not fix.

- Skip silently on any extension Biome does not handle (allowlist: `.js .jsx .mjs .cjs .ts .tsx .mts .cts .json .jsonc .css`).
- Spawn with `cwd: process.env.CLAUDE_PROJECT_DIR ?? process.cwd()` — the hook may be invoked from anywhere.
- On unfixable findings: write them to stderr, then `process.exit(2)` under Claude (that is how hook output reaches the model) and `0` under Cursor.

## harness-score L4

Local equivalent of the CI job:

```bash
npx --yes harness-score@1.6.3 . --min-level 4
```

Reaching L4 needs all of the following present and non-trivial:

- `AGENTS.md` at the root — layout table, command list, conventions, verification.
- A per-area `AGENTS.md` in each project that has rules of its own (`apps/api/`, `apps/web/`, `packages/db/`).
- `.agents/rules/` — numbered, scoped, `alwaysApply: true`.
- `.agents/skills/` — at least the workflows an agent repeats: migrations, adding an endpoint, verifying changes, QA against a running stack.
- `.agents/workflows/` — slash commands that invoke those skills.
- `.agents/agents/` — reviewer and QA subagents with an explicit `tools` list. Give a subagent the work whose tool output is bulky and whose result is small; that is what keeps the parent context clean.
- Both hooks wired, in both tools.
- CI enforcing the score, so it cannot silently regress.

Run it before declaring the scaffold finished, and paste the real output.
