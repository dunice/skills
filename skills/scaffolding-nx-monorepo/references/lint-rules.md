# Lint rules

Biome is the only linter and the only formatter. Everything below is enforced by `pnpm check`, which is what the husky hooks and CI both run — so a rule that only warns is a rule nobody obeys.

Two facts shape this whole file:

- **Most shape rules ship at `info` or `warn`.** They print, exit 0, and change nothing. Every rule here pins `"level": "error"` explicitly.
- **Overrides apply in array order and the last one wins.** The `**/*.ts` + `**/*.tsx` override carries the shape rules, so it must come *before* the narrower `apps/api/**` and spec-file overrides that switch individual rules back off.

## What is enforced

| Rule | Setting | Scope |
|---|---|---|
| Type-only imports first, in their own block | `organizeImports` group `{ "type": true }` + `useImportType: separatedType` | `.ts`, `.tsx` |
| Imports grouped node → third-party → project | `organizeImports.groups` | `.ts`, `.tsx` |
| Imports sorted alphabetically inside each group | `identifierOrder` (default `natural`) | `.ts`, `.tsx` |
| Blank line between import groups | `":BLANK_LINE:"` separators | `.ts`, `.tsx` |
| Max 50 lines per function | `complexity/noExcessiveLinesPerFunction` | `.ts`, `.tsx` |
| Max 2 levels of nested `if`/`else` | GritQL plugin | `.ts`, `.tsx` |
| Max 1000 lines per file | `style/noExcessiveLinesPerFile` | `.ts`, `.tsx` |
| No `else` after an early return | `style/noUselessElse` | `.ts`, `.tsx` |
| Braces on every `if` / `else` body | `style/useBlockStatements` | `.ts`, `.tsx` |
| A braced body never sits on one line | the formatter — no rule needed | everywhere |
| Max 3 parameters, then take an object | `complexity/useMaxParams` | `.ts`, `.tsx` |
| An object literal past one property spans lines | GritQL plugin | `.ts`, `.tsx` |
| Naming conventions | `style/useNamingConvention` | `.ts`, `.tsx` |
| No `console`, any method | `suspicious/noConsole: "error"` | everywhere |
| Hooks called unconditionally | `correctness/useHookAtTopLevel` | `apps/web/**` |
| Nest DI keeps runtime imports | `useImportType: "off"` | `apps/api/**` |
| `console` allowed in tests | `suspicious/noConsole: "off"` | `*.spec.*`, `*.test.*` |
| Env schema keys keep their real names | `useNamingConvention: "off"` | `apps/api/src/env.ts` |

## If you were handed ESLint rule names

There is no ESLint in this repo and there is not going to be one — property 5 of the contract. Requests still arrive in ESLint's vocabulary, because that is the vocabulary everyone has. This is the translation:

| Asked for, in ESLint terms | What actually enforces it here |
|---|---|
| `curly` | `style/useBlockStatements` |
| `brace-style`, `max-statements-per-line` | the formatter — it breaks any braced body across lines on its own |
| `padding-line-between-statements` | **nothing** — no Biome rule, and GritQL cannot express it. See "When a rule needs a script" |
| `max-params` | `complexity/useMaxParams` |
| `object-curly-newline` / `object-property-newline` | `scripts/biome-plugins/no-single-line-multi-prop-object.grit` |
| `max-depth` | `scripts/biome-plugins/no-deep-if-nesting.grit` |
| `max-lines`, `max-lines-per-function` | `style/noExcessiveLinesPerFile`, `complexity/noExcessiveLinesPerFunction` |
| `no-console` | `suspicious/noConsole` |
| `no-else-return` | `style/noUselessElse` |
| `@typescript-eslint/naming-convention` | `style/useNamingConvention` |
| `sort-imports` / `import/order` | the `organizeImports` assist action |
| `react-hooks/rules-of-hooks` | `correctness/useHookAtTopLevel` |

Two of these have no Biome rule and are carried by a GritQL plugin instead. One has nothing at all. Reach for a plugin only after checking that no rule exists, and for a script only after checking that GritQL cannot express it either — the list of what GritQL can express is under "Writing GritQL for Biome".

## Imports

One `organizeImports` config does grouping, ordering and blank lines at once. It is an **assist action**, not a lint rule, so `pnpm check` reports it and `pnpm check:fix` applies it.

Before:

```ts
import { bla, type Schema } from 'zod';
import { baz, type Pool } from 'pg';
import type { Database } from '#db';
import { readFile } from 'node:fs/promises';
import { trpc } from '../lib/trpc.js';
```

After:

```ts
import type { Pool } from 'pg';
import type { Schema } from 'zod';
import type { Database } from '#db';

import { readFile } from 'node:fs/promises';

import { baz } from 'pg';
import { bla } from 'zod';

import { trpc } from '../lib/trpc.js';
```

Four things happened:

1. `{ type: true }` is the first group, so every type-only import rises to the top regardless of where it resolves from.
2. `useImportType` with `style: "separatedType"` split `import { bla, type Schema }` into a value import and an `import type`. Without it, an inline `type` specifier stays welded to its value import and never reaches the type block.
3. `:NODE:` / `:PACKAGE:` / `:ALIAS:` + `:PATH:` produce the three value groups. `:ALIAS:` is what catches the `#contracts` / `#db` subpath imports; `:PATH:` catches relative ones.
4. `":BLANK_LINE:"` between each group inserts the separating line.

Two settings deliberately left alone:

- `sortBareImports` stays `false`. Turn it on and `import 'reflect-metadata'` sorts away from line 1 of `apps/api/src/main.ts`; Nest's decorator metadata never registers and DI fails at runtime with no lint error.
- `identifierOrder` stays `natural`, which sorts `item2` before `item10`.

## Function and file size

```ts
// complexity/noExcessiveLinesPerFunction — counts the function BODY only
export function tooLong() {
  // 51 body lines → × This function has too many lines (52). Maximum allowed is 50.
}
```

`skipBlankLines: false` and `skipIifes: false` are set explicitly rather than left to default, so the number in the config is the number a reader counts in the editor.

`style/noExcessiveLinesPerFile` at `maxLines: 1000` counts physical lines including blanks. Generated files never reach it: `packages/db/migrations` and `**/dist` are already outside `files.includes`.

## Naming

`style/useNamingConvention` ships at `info` with no conventions, which checks almost nothing. The scaffold spells out one convention per declaration kind. **Conventions are evaluated in order and the first matching selector wins**, so the narrow `const` + `scope: "global"` entry has to precede the general `const` one.

```json
"useNamingConvention": {
  "level": "error",
  "options": {
    "strictCase": true,
    "requireAscii": true,
    "conventions": [
      { "selector": { "kind": "typeAlias" }, "formats": ["PascalCase"] },
      { "selector": { "kind": "objectLiteralProperty" }, "formats": ["camelCase"] },
      { "selector": { "kind": "functionParameter" }, "formats": ["camelCase"] },
      { "selector": { "kind": "const", "scope": "global" }, "match": "(?:[a-z]|.{3,30})", "formats": ["camelCase", "CONSTANT_CASE"] },
      { "selector": { "kind": "const" }, "match": "(?:[a-z]|.{3,30})", "formats": ["camelCase"] },
      { "selector": { "kind": "let" }, "match": "(?:[a-z]|.{3,30})", "formats": ["camelCase"] }
    ]
  }
}
```

`strictCase: true` is what bans two consecutive uppercase letters — `HTTPServer` fails, `HttpServer` passes. It is a single switch, not a separate rule.

The `match` regex carries the length rule. `match` and `formats` both apply: the name must satisfy the regex *and* the case format. `(?:[a-z]|.{3,30})` reads as "a single lowercase letter, or between three and thirty characters", which keeps `for (let i = 0; …)` legal while rejecting `ab` and anything runaway.

The regex is **anchored to the whole name** — Biome does not match it as a substring — which is what makes the upper bound work at all. Verified at the edges: 30 characters passes, 31 fails.

**The single-letter allowance is wider than it looks.** Biome's selectors cannot see that a variable belongs to a `for` initialiser, so any single lowercase letter passes anywhere, not just in loops. This is the closest the rule can get; there is no loop-scoped selector.

**Pick the ceiling against the codebase, not in the abstract.** A `<thing>Schema` convention spends six characters on the suffix before naming anything, so a contracts package hits a low ceiling fast: at 20, `createCandidateInputSchema` (26) and `interviewStatusSchema` (21) both fail, and the fix is renaming the public cross-project API. 30 leaves those intact and still rejects a 40-character name.

What each entry produces:

```ts
type UserRecord = { … };          // ✓   type ProfileHTTPData → ✗ strictCase
const MAX_RETRIES = 3;            // ✓   global const, CONSTANT_CASE allowed
const apiTimeout = 500;           // ✓   global const, camelCase also allowed
function scan(items: string[]) {  // ✓   parameter camelCase
  let total = 0;                  // ✓   within 3–30
  for (let i = 0; i < items.length; i++) { … }   // ✓   single letter
  const ab = 1;                   // ✗   const name should match /(?:[a-z]|.{3,30})/
  const conf = { Some_Key: 1 };   // ✗   object property should be in camelCase
}
```

Two consequences worth budgeting for before switching this on:

- **A zod env schema cannot satisfy it.** `NODE_ENV`, `DATABASE_URL` and friends are `process.env` keys, not identifiers, and renaming them breaks the app. `apps/api/src/env.ts` gets an override turning the rule off — the only place in the config where a rule is disabled because the code is right and the rule is wrong.
- **Two-character names have to go.** The floor renamed `const db` → `database` in `apps/api/src/main.ts` and `packages/db/src/client.ts`. The exported property keys stayed `db`, so `Database = ReturnType<typeof createDatabase>['db']` still resolves. A three-character floor leaves `const app` alone; a four-character one does not.

### Class member modifier order

There is no Biome rule for this, and none is needed. TypeScript enforces it in the compiler:

```
error TS1029: 'private' modifier must precede 'static' modifier.
```

`pnpm typecheck` already fails on it.

## console

`suspicious/noConsole` is set to `"error"` with no `allow` list — no method is exempt, including `console.error` and `console.info`. Spec files override it back off.

A CLI that legitimately prints writes to the stream directly:

```ts
process.stderr.write('DATABASE_URL is not set. Copy .env.example to .env first.\n');
process.stdout.write(`Migrations applied from ${migrationsFolder}\n`);
```

Note the explicit `\n` — `write` does not append one.

## Branch shape

Two mechanisms, one shape:

```ts
if (foo) return bar;              // ✗ style/useBlockStatements
if (foo) { return bar; }          // ✗ the formatter rewrites it
if (foo) {                        // ✓
  return bar;
}
```

`style/useBlockStatements` covers the braces. Its fix is classed **unsafe** — dropping a statement into a block can change what a comment attaches to — so `biome check --write` leaves it alone and an existing codebase is migrated once with:

```bash
pnpm exec biome lint --write --unsafe --only=style/useBlockStatements .
```

The one-line body needs no rule at all. Biome's formatter always breaks a braced body across lines, so `if (a) { return 1; }` is a *formatting* failure and `pnpm check` already gates on it.

## Argument count

`complexity/useMaxParams` at `max: 3`. A fourth argument means an options object:

```ts
function ask(session, job, turn, answer) { … }     // ✗ Function has 4 parameters, but only 3 are allowed
function ask({ session, job, turn, answer }) { … } // ✓ one parameter
```

A destructured object counts as one parameter, which is the whole point. Give it a named interface rather than an inline type literal once it has more than two or three fields — the call sites read better and the type is reusable.

Nest constructor injection is the case to check before turning this on: a service with four `@Inject()` parameters fails, and the fix there is fewer collaborators, not a bigger ceiling.

## Object literal shape

One property may sit inline. Two may not:

```ts
{ text: question }                                              // ✓
{ text: question, slotId: slot.id, kind: 'planned' }            // ✗
{
  text: question,
  slotId: slot.id,
  kind: 'planned',
}                                                                // ✓
```

No Biome rule does this. `formatter.expand: "always"` is the closest switch and it is wrong in both directions — it expands single-property objects too, and it expands arrays. So it is a second GritQL plugin, `scripts/biome-plugins/no-single-line-multi-prop-object.grit`:

```grit
language js

`{ $props }` as $object where {
  $object <: r"\{[^\n]*\}",
  $props <: [$_, $_, ...],
  register_diagnostic(
    span=$object,
    message="Object literal with more than one property must span multiple lines — put each property on its own line."
  )
}
```

Two things make it precise:

- `` `{ $props }` `` binds object **expressions** only. Import clauses, destructuring patterns and TS type literals parse as other nodes and never reach the pattern — verified against a fixture holding all three.
- The property count comes from the list pattern `[$_, $_, ...]`, not from counting commas in the source. `{ fn: Math.max(1, 2) }` has a comma and one property, and is not flagged.

The regex is what asks "is this on one line", and it works because a node's text is its own source, without trivia.

To migrate an existing codebase, insert a newline after the `{` of every offender and let the formatter finish the job — `expand: "auto"` (the default) expands any object whose first property has a leading newline, and indents the rest.

## Nesting depth

Biome has no `max-depth` equivalent, so this one is a GritQL plugin at `scripts/biome-plugins/no-deep-if-nesting.grit`:

```grit
language js

// Flags an `if` nested three deep. Nesting is counted through the bodies an
// `if` owns — its consequence block and its `else { … }` block — so an
// `if / else if / else if` chain stays at depth 1, the way ESLint's `max-depth`
// treats it. Only the offending innermost `if` is reported.
pattern if_body($body) {
  or {
    `if ($_) { $body }`,
    `if ($_) { $_ } else { $body }`
  }
}

if_body($outer) where {
  $outer <: contains if_body($middle) where {
    $middle <: contains `if ($_) { $_ }` as $deep where {
      register_diagnostic(
        span = $deep,
        message = "Too many nested if statements (3). Maximum nesting depth is 2 — use early returns or extract the inner branch into a function."
      )
    }
  }
}
```

What it does and does not flag:

```ts
if (a) { if (b) { if (c) { … } } }                 // ✗ flagged, depth 3
if (a) { return 1; } else { if (b) { if (c) { … } } }  // ✗ flagged, else block counts
if (a) { if (b) { … } }                            // ✓ depth 2
if (a) { … } else if (b) { … } else if (c) { … }   // ✓ a chain, not nesting
```

Plugin diagnostics are errors and exit 1, so the hooks and CI gate on them with no extra wiring.

### Writing GritQL for Biome

Biome's GritQL vocabulary is much narrower than the upstream language, and a pattern that fails to match is indistinguishable from a pattern that works. Probe every new pattern against a fixture before trusting it.

| Form | Result |
|---|---|
| `` `if ($c) { $body }` `` | works — braces are required |
| `` `if ($c) $body` `` | compiles, matches nothing |
| `if_statement(consequence = $c)` | works |
| `if_statement(alternative = $a)` | fails to compile |
| `statement_block()`, `else_clause()` | fail to compile |
| `or { … }`, `pattern name($x) { … }`, `contains`, `as` | work |
| `` $x <: after `…` `` | works — and means the *immediate* preceding sibling |
| `$x <: within $outer` | works, and binds an ancestor |
| `$list <: [$_, $_, ...]` and `$list <: [..., <snippet>, $next, ...]` | work — element counting and positional matching over a node's list |
| `$node <: r"…"` | works — the regex runs against the node's own source text |
| `` `if ($c) { $b } $next` `` (a two-statement snippet) | compiles, matches nothing |
| `$name` interpolated into `r"…"` | compiles, matches nothing (the same pattern with a literal matches) |
| `before` | did not match in any probe |

The trivia rule is what decides most of this. A **statement** node's text excludes leading and trailing trivia, so nothing about the whitespace around it is readable. A **list** node's text does include the gaps between its elements — but a regex over the list cannot be tied to the element the list pattern matched. Anything about *whitespace between two specific siblings* is therefore out of reach; see "When a rule needs a script" for the worked example.

To confirm a plugin is loaded at all, point `plugins` at a path that does not exist — Biome then reports `Error(s) during loading of plugins`. Silence means it loaded and simply matched nothing.

## Editor integration

A rule nobody sees until commit time is half a rule, so scaffold this in the same pass.

Biome's LSP publishes everything the CLI does — lint rules, assist actions and **GritQL plugin diagnostics**, which arrive with the source `biome/plugin`. Nothing extra is needed to make a plugin visible; confirmed by driving `biome lsp-proxy` directly rather than trusting the docs.

- `.vscode/extensions.json` — recommend `biomejs.biome`, and list the ESLint and Prettier extensions under `unwantedRecommendations`. Both are installed on most machines and both will fight Biome.
- `.vscode/settings.json` — **commit it.** Biome as `editor.defaultFormatter` for every language it owns, `source.fixAll.biome` and `source.organizeImports.biome` in `codeActionsOnSave`, `prettier.enable: false`, `eslint.enable: false`. The usual `.gitignore` line is `.vscode/*` plus a negation per file worth sharing.
- Never put `quickfix.biome` in `codeActionsOnSave`. Biome's own `suspicious/noQuickfixBiome` flags it: it applies every rule's fix atomically, and two fixes over the same span emit invalid code. Use `source.fixAll.biome`.
- **Cursor does not inherit VS Code's installed extensions.** A repo whose diagnostics "do not show up" is usually a Cursor session with no Biome extension. Check before debugging the config.

## When a rule needs a script

Some rules are expressible by neither a Biome rule nor a GritQL plugin. Before writing a script, work down this list — and know that the last step costs more than it looks.

**1. Confirm no rule exists.** Check the installed schema, not the documentation:

```bash
node -e "const d=require('./node_modules/@biomejs/biome/configuration_schema.json').definitions;
  console.log(['A11y','Complexity','Correctness','Nursery','Performance','Security','Style','Suspicious']
    .flatMap(g=>Object.keys(d[g].properties).map(k=>g+'/'+k))
    .filter(r=>/<your keyword>/i.test(r)).join('\n'))"
```

**2. Confirm GritQL cannot express it.** The vocabulary table under "Writing GritQL for Biome" is the reference. The trap is that a plugin can *look* like it works.

Worked example — a rule requiring a blank line after every `if`/`else` block. This repo carried one for a while and then removed it, and the reason is instructive. No Biome rule covers blank lines (540 rules, none match), and the formatter has no key for them. A plugin compiles and matches:

```grit
$list where {
  $list <: [..., `if ($_) { $_ }`, $next, ...],
  $list <: r"(?s).*\}\n[ \t]*[^\s\n].*",
  register_diagnostic(span=$next, message="…")
}
```

It is also wrong in both directions at once. The regex is list-wide and cannot be tied to the pair the list pattern found, and it cannot tell an `if`'s closing brace from a `for`'s, a nested function's or a `try`'s:

```ts
export function falsePositive(a: number) {
  if (a) {
    return 1;
  }

  for (const x of [1, 2]) {     // ← flagged, and there is nothing wrong here
    process.stdout.write(`${x}`);
  }
  return 2;                     // ← the real gap, not an if, not reported
}
```

The one construct that would tie the regex to the pair — interpolating `$next` into it — compiles and matches nothing.

**3. Write the script, and wire it everywhere Biome is wired.** A script in `package.json` alone is invisible to `nx affected -t lint` and to the pre-push hook:

```jsonc
"lint":      "biome lint . && node scripts/<rule>.mjs .",
"check":     "biome check . && node scripts/<rule>.mjs .",
"check:fix": "biome check --write . && node scripts/<rule>.mjs --write .",

"lint-staged": {
  "*.{ts,mts,cts,tsx}": ["node scripts/<rule>.mjs --write"]
}
```

```jsonc
// every project.json
"lint": { "executor": "nx:run-commands", "options": { "command": "biome lint apps/api && node scripts/<rule>.mjs apps/api" } }
```

Biome runs first in each pair, so a fix that only inserts whitespace cannot un-format anything.

**4. Give it a TypeScript language-service plugin, or it stays invisible in the editor.** Share one implementation so the editor and the gate cannot drift:

```
scripts/lib/<rule>.cjs                                   the rule, findViolations(ts, sourceFile)
scripts/<rule>.mjs                                       the CLI, imports it
scripts/tsserver-plugins/node_modules/<rule>/index.cjs   the editor plugin, requires it
scripts/tsserver-plugins/node_modules/<rule>/package.json
```

The plugin proxies `getSemanticDiagnostics` and appends its own, using the `ts` instance tsserver hands it — never a `require('typescript')` of its own, or the node predicates fail across realms.

That directory shape is forced. Four facts, each read out of tsserver's resolver and then confirmed against a real run:

| Fact | Consequence |
|---|---|
| `requestEnablePlugin` rejects any name that is relative or contains `..` — *"only package name is allowed plugin name"* | The plugin cannot be referenced by path |
| Plugins resolve as `<probe location>/node_modules/<name>` | A literal `node_modules` directory is mandatory |
| Resolution tries only `package.json`, `.js`, `.jsx`, `index.js`, `index.jsx` | A bare `index.cjs` is never found |
| The root manifest is `"type": "module"`, so an `index.js` there is ESM and `require()` fails | A `package.json` with `"type": "commonjs"` and `"main": "index.cjs"` is the only way through |

That manifest sits **inside** `node_modules`, so pnpm and Nx never see it and the contract's own check — `find . -name package.json -not -path '*/node_modules/*'` — still returns one line. `.gitignore` needs a matching exception, and the directory must be un-excluded before its contents can be:

```gitignore
node_modules/
!scripts/tsserver-plugins/node_modules/
```

Then `tsconfig.base.json` carries `"plugins": [{ "name": "<rule>" }]` (inherited by every project; `tsc` ignores it) and `.vscode/settings.json` carries `"typescript.tsserver.pluginPaths": ["./scripts/tsserver-plugins"]`.

To verify rather than hope, run tsserver headless with `--pluginProbeLocations <dir> --logVerbosity verbose --logFile <path>`, send `open` then `geterr`, and look for `Plugin validation succeeded` in the log and your diagnostic in the `semanticDiag` event. A failed load prints every path it tried, which is how the four facts above were established.

**Then weigh it.** One rule that Biome cannot express costs a shared core, a CLI, an editor plugin, a manifest inside `node_modules`, two `.gitignore` exceptions, a `tsconfig` entry, an editor setting and a line in four `package.json` scripts plus three `project.json` targets — every one of which is a place for the wiring to rot. The blank-line rule above was built exactly this way and then removed, because that is a lot of surface for one blank line. Ship the rule if it earns its keep; the recipe is here either way.

## biome.json

The scaffold's config in full. Versions and the ignore list will differ; the structure should not.

**Copy it as-is — no comments.** A single `//` line anywhere in `biome.json` makes Biome discard the entire config and silently fall back to its defaults: no parse error, no warning, just single quotes turning into double ones on the next format. (`json.parser.allowComments` governs the `.json` files Biome *lints*, not its own config.) The three annotations that used to sit inline are listed under the block instead.

```json
{
  "$schema": "https://biomejs.dev/schemas/<exact biome version>/schema.json",
  "vcs": {
    "enabled": true,
    "clientKind": "git",
    "useIgnoreFile": true
  },
  "files": {
    "includes": [
      "**",
      "!**/dist",
      "!**/coverage",
      "!**/node_modules",
      "!**/.nx",
      "!pnpm-lock.yaml",
      "!packages/db/migrations",
      "!.agents/skills/impeccable"
    ]
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100,
    "lineEnding": "lf"
  },
  "linter": {
    "enabled": true,
    "rules": {
      "preset": "recommended",
      "suspicious": {
        "noConsole": "error"
      }
    }
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "single",
      "semicolons": "always",
      "trailingCommas": "all",
      "arrowParentheses": "always"
    },
    "parser": {
      "unsafeParameterDecoratorsEnabled": true
    }
  },
  "json": {
    "parser": {
      "allowComments": true
    }
  },
  "assist": {
    "actions": {
      "source": {
        "organizeImports": "on"
      }
    }
  },
  "overrides": [
    {
      "includes": [
        "**/*.ts",
        "**/*.tsx"
      ],
      "plugins": [
        "scripts/biome-plugins/no-deep-if-nesting.grit",
        "scripts/biome-plugins/no-single-line-multi-prop-object.grit"
      ],
      "assist": {
        "actions": {
          "source": {
            "organizeImports": {
              "level": "on",
              "options": {
                "groups": [
                  {
                    "type": true
                  },
                  ":BLANK_LINE:",
                  [
                    ":NODE:",
                    ":BUN:"
                  ],
                  ":BLANK_LINE:",
                  ":PACKAGE:",
                  ":BLANK_LINE:",
                  [
                    ":ALIAS:",
                    ":PATH:"
                  ]
                ]
              }
            }
          }
        }
      },
      "linter": {
        "rules": {
          "style": {
            "noExcessiveLinesPerFile": {
              "level": "error",
              "options": {
                "maxLines": 1000
              }
            },
            "useImportType": {
              "level": "error",
              "options": {
                "style": "separatedType"
              }
            },
            "noUselessElse": {
              "level": "error"
            },
            "useBlockStatements": {
              "level": "error"
            },
            "useNamingConvention": {
              "level": "error",
              "options": {
                "strictCase": true,
                "requireAscii": true,
                "conventions": [
                  {
                    "selector": {
                      "kind": "typeAlias"
                    },
                    "formats": [
                      "PascalCase"
                    ]
                  },
                  {
                    "selector": {
                      "kind": "objectLiteralProperty"
                    },
                    "formats": [
                      "camelCase"
                    ]
                  },
                  {
                    "selector": {
                      "kind": "functionParameter"
                    },
                    "formats": [
                      "camelCase"
                    ]
                  },
                  {
                    "selector": {
                      "kind": "const",
                      "scope": "global"
                    },
                    "match": "(?:[a-z]|.{3,30})",
                    "formats": [
                      "camelCase",
                      "CONSTANT_CASE"
                    ]
                  },
                  {
                    "selector": {
                      "kind": "const"
                    },
                    "match": "(?:[a-z]|.{3,30})",
                    "formats": [
                      "camelCase"
                    ]
                  },
                  {
                    "selector": {
                      "kind": "let"
                    },
                    "match": "(?:[a-z]|.{3,30})",
                    "formats": [
                      "camelCase"
                    ]
                  }
                ]
              }
            }
          },
          "complexity": {
            "useMaxParams": {
              "level": "error",
              "options": {
                "max": 3
              }
            },
            "noExcessiveLinesPerFunction": {
              "level": "error",
              "options": {
                "maxLines": 50,
                "skipBlankLines": false,
                "skipIifes": false
              }
            }
          }
        }
      }
    },
    {
      "includes": [
        "apps/web/**"
      ],
      "linter": {
        "rules": {
          "correctness": {
            "useHookAtTopLevel": "error"
          }
        }
      }
    },
    {
      "includes": [
        "apps/api/**"
      ],
      "linter": {
        "rules": {
          "style": {
            "useImportType": "off"
          }
        }
      }
    },
    {
      "includes": [
        "**/*.spec.ts",
        "**/*.spec.tsx",
        "**/*.test.ts",
        "**/*.test.tsx"
      ],
      "linter": {
        "rules": {
          "suspicious": {
            "noConsole": "off"
          }
        }
      }
    },
    {
      "includes": [
        "apps/api/src/env.ts"
      ],
      "linter": {
        "rules": {
          "style": {
            "useNamingConvention": "off"
          }
        }
      }
    }
  ]
}
```

The three things worth knowing about that block, which cannot be written into it:

- `javascript.parser.unsafeParameterDecoratorsEnabled` is there for Nest's `@Inject()` parameter decorators.
- `overrides` order matters: broad shape rules first, narrow exemptions after.
- The `apps/api/**` exemption exists because a type-only import emits no runtime value, so Nest's `design:paramtypes` metadata goes `undefined` and constructor injection breaks — silently.

## Traps

| Trap | Symptom | Fix |
|---|---|---|
| Shape rules left at their default severity | `noExcessiveLinesPerFile` / `PerFunction` print but never fail a hook or CI | Pin `"level": "error"` on each |
| `null` used as an import-group separator | The config schema advertises it, the runtime rejects it, Biome refuses to start | Use the `":BLANK_LINE:"` string |
| `apps/api/**` override placed before the shape override | Nest constructor injection returns `undefined`, no lint error | Broad override first, exemptions after |
| `sortBareImports: true` | `reflect-metadata` moves off line 1; Nest DI dies at runtime | Leave it `false` |
| A comment in `biome.json` | Whole config discarded, defaults applied, no error — quote style and every rule silently revert | Keep the config pure JSON |
| `noConsoleLog` copied from a 1.x config | `Found an unknown key` — Biome refuses to start | Renamed to `noConsole` in 1.6.0 |
| `"recommended": true` copied from a 1.x config | Deprecated-field diagnostic, removal in the next major | Use `"preset": "recommended"` |
| `noUselessElse` filed under `complexity` | `Found an unknown key` — it lives in `style` | Check the group with `biome explain <rule>` |
| A length ceiling picked without checking the codebase | Exported schema/contract names blow the limit; the "fix" is renaming public API | Measure the longest existing identifiers first |
| `useNamingConvention` conventions in the wrong order | The general `const` entry shadows the `scope: "global"` one, so `CONSTANT_CASE` is rejected | Narrowest selector first |
| A GritQL pattern that silently matches nothing | Rule appears to pass on code that violates it | Probe against a fixture; check load errors with a deliberately bad plugin path |
| `useBlockStatements` expected to auto-fix | `biome check --write` reports it and changes nothing | Its fix is unsafe; migrate once with `biome lint --write --unsafe --only=style/useBlockStatements .` |
| A GritQL pattern written to check whitespace between siblings | Compiles, matches, and reports the wrong node | Not expressible — see "When a rule needs a script" before writing one |
| A lint script wired into `package.json` only | `nx affected -t lint` and the pre-push hook skip it entirely | Append it to every `project.json` `lint` target too |
| Expanding object literals across lines after the fact | Functions that were at 48 lines cross the 50-line ceiling | Re-run `biome lint` after the migration and split what tipped over |
| `useMaxParams` turned on with Nest DI in the tree | Constructors with four `@Inject()` parameters fail | Real fix is fewer collaborators; the ceiling is not the problem |

## Verifying

A fixture per rule, run through `pnpm exec biome lint <file>`, is the only proof that a rule is live. The scaffold is not done until each of these fails:

- a file with a 3-deep `if`
- a function with 51 body lines
- a file with 1001 lines
- a file with `import { a, type B } from 'x'` below a value import
- a `type httpData` alias, a `HTTPServer` identifier, a `const ab`, a 31-character const, an object key `Some_Key`, a `function f(On: boolean)`
- a React component calling a hook inside an `if`
- any `console.*` call outside a spec file
- an `else` after a branch that returns
- `if (foo) return bar;` — no braces
- `if (foo) { return bar; }` — a braced body on one line, which fails `biome format` rather than `biome lint`
- a function with four parameters
- `const many = { a, b, c };` — two or more properties on one line

And these must pass: `for (let i = 0; …)`, a global `const MAX_RETRIES`, a global `const apiTimeout`, `const one = { text: 'q' };`, `import { a, b } from 'x'`, `const { a, b } = one;` and `{ fn: Math.max(1, 2) }`.

Then delete the fixtures.
