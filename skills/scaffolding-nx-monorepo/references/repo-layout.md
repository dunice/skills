# Repo layout and build wiring

Every path is relative to the repository root, and **every command runs from the root** — `drizzle.config.ts` and `vitest.config.ts` both assume it.

## Tree

```
.agents/          skills/ workflows/ agents/ rules/     (the harness; see harness-and-hooks.md)
.claude/          settings.json + symlinks into .agents/
.cursor/          hooks.json — nothing else
.github/workflows/ci.yml, harness-score.yml
.husky/           pre-commit, pre-push
apps/api/         NestJS: HTTP, DI, config, the tRPC router
apps/web/         React + Vite, typed tRPC client
packages/contracts/  Zod schemas shared by both sides
packages/db/      Drizzle schema, migrations, pooled client factory
scripts/          hook scripts + repo scripts; biome-plugins/ holds the GritQL lint plugins
AGENTS.md         the map; CLAUDE.md is a symlink to it
package.json      the only manifest
```

Dependency direction is one-way: `contracts` ← `db` ← `api` ← `web`. The web app imports only the `AppRouter` **type** from the api.

## Root manifest

```jsonc
{
  "name": "<repo>",
  "private": true,
  "type": "module",
  "packageManager": "pnpm@<exact>",
  "engines": { "node": ">=<nvmrc>", "pnpm": ">=<major>" },

  // Cross-project resolution. No path aliases, no relative reaching.
  // These point at BUILD OUTPUT — hence dependsOn: ["^build"] everywhere.
  "imports": {
    "#contracts":   { "types": "./packages/contracts/dist/index.d.ts", "default": "./packages/contracts/dist/index.js" },
    "#db":          { "types": "./packages/db/dist/index.d.ts",        "default": "./packages/db/dist/index.js" },
    "#api-contract":{ "types": "./apps/api/dist/trpc/index.d.ts",      "default": "./apps/api/dist/trpc/index.js" }
  },

  "scripts": {
    "prepare":      "husky",
    "dev":          "nx run-many -t serve -p api web",
    "build":        "nx run-many -t build",
    "test":         "nx run-many -t test",
    "test:watch":   "vitest",
    "typecheck":    "nx run-many -t typecheck",
    "lint":         "biome lint .",
    "format:check": "biome format .",
    "check":        "biome check .",
    "check:fix":    "biome check --write .",
    "db:generate":  "drizzle-kit generate --config packages/db/drizzle.config.ts --name",
    "db:migrate":   "nx run db:migrate",
    "db:studio":    "drizzle-kit studio --config packages/db/drizzle.config.ts",
    "docker:up":    "docker compose up -d --wait",
    "docker:down":  "docker compose down",
    "docker:test:up":   "docker compose -f compose.test.yml up -d --wait",
    "docker:test:down": "docker compose -f compose.test.yml down -v",
    "harness:score":"npx --yes harness-score@1.6.3 . --min-level 4"
  },

  "lint-staged": {
    "*.{js,mjs,cjs,jsx,ts,mts,cts,tsx,json,jsonc,css}": [
      "biome check --write --no-errors-on-unmatched"
    ]
  }
}
```

`db:generate` ends in `--name` on purpose: `pnpm db:generate add_scores` appends the argument, so a named migration is the only thing the script can produce.

## nx.json

```jsonc
{
  "namedInputs": {
    "default": ["{projectRoot}/**/*", "sharedGlobals"],
    "production": ["default", "!{projectRoot}/**/*.spec.ts", "!{projectRoot}/**/*.spec.tsx", "!{projectRoot}/**/*.md"],
    "sharedGlobals": [
      "{workspaceRoot}/tsconfig.base.json",
      "{workspaceRoot}/biome.json",
      "{workspaceRoot}/package.json",
      "{workspaceRoot}/vitest.config.ts"
    ]
  },
  "targetDefaults": {
    "build":     { "dependsOn": ["^build"], "inputs": ["production", "^production"], "outputs": ["{projectRoot}/dist"], "cache": true },
    "typecheck": { "dependsOn": ["^build"], "inputs": ["default", "^production"], "cache": true },
    "test":      { "dependsOn": ["^build"], "inputs": ["default", "^production"], "outputs": ["{projectRoot}/coverage"], "cache": true },
    "lint":      { "inputs": ["default"], "cache": true },
    "serve":     { "dependsOn": ["^build"], "cache": false }
  },
  "defaultBase": "origin/main"
}
```

`defaultBase` is what makes `nx affected` in the hooks and in CI scope correctly without a flag at every call site.

## project.json — the shape

There are no per-project manifests and no Nx plugins. Every target is `nx:run-commands` invoking the tool directly, from the root.

```jsonc
{
  "$schema": "../../node_modules/nx/schemas/project-schema.json",
  "name": "api",
  "projectType": "application",           // "library" for packages/*
  "sourceRoot": "apps/api/src",
  "implicitDependencies": ["contracts", "db"],   // the graph, stated explicitly
  "targets": {
    "build":     { "executor": "nx:run-commands", "options": { "command": "tsc -p apps/api/tsconfig.json" } },
    "typecheck": { "executor": "nx:run-commands", "options": { "command": "tsc -p apps/api/tsconfig.typecheck.json" } },
    "test":      { "executor": "nx:run-commands", "options": { "command": "vitest run --project api" } },
    "lint":      { "executor": "nx:run-commands", "options": { "command": "biome lint apps/api" } },
    "serve": {
      "executor": "nx:run-commands",
      "dependsOn": ["build"],
      "options": {
        "parallel": true,
        "commands": [
          "tsc -p apps/api/tsconfig.json --watch --preserveWatchOutput",
          "node --watch apps/api/dist/main.js"
        ]
      }
    }
  }
}
```

Because `implicitDependencies` is declared by hand, adding a cross-project import means editing the consuming `project.json` in the same commit. Skip it and `nx affected` will miss the dependent project.

### The web build target — the NODE_ENV trap

Nx loads the root `.env` into **every** task environment. A local `.env` with `NODE_ENV=development` therefore leaks into `vite build`, and Vite ships the development build of React with no warning at all. Pin it on the target:

```jsonc
"build": {
  "executor": "nx:run-commands",
  "options": {
    "command": "vite build --config apps/web/vite.config.ts",
    "env": { "NODE_ENV": "production" }
  }
}
```

### db migrate target

```jsonc
"migrate": {
  "executor": "nx:run-commands",
  "dependsOn": ["build"],
  "cache": false,                              // never cache a side effect
  "options": { "command": "node packages/db/dist/migrate.js" }
}
```

## TypeScript

ESM everywhere. Relative imports in `apps/api` and `packages/*` carry the `.js` extension even from `.ts` sources — that is a requirement of Node ESM, not a style choice.

- `tsconfig.base.json` — strictness and emit options only (`strict`, `noUncheckedIndexedAccess`, `isolatedModules`, `declaration`, `declarationMap`, `sourceMap`). **No `module`/`moduleResolution`** — those differ per project and belong in the leaf config.
- `<project>/tsconfig.json` — build config: `outDir: ./dist`, `rootDir: ./src`, and `"exclude": ["**/*.spec.ts"]` so test code never ships.
- `<project>/tsconfig.typecheck.json` — extends the build config with `noEmit: true` and an `include` that adds the specs back. Two files because the build must not emit test code but typecheck must still see it.

Per-project module settings:

| Project | `module` | `moduleResolution` | Extra |
|---|---|---|---|
| `packages/*`, `apps/api` | `nodenext` | `nodenext` | api adds `experimentalDecorators`, `emitDecoratorMetadata`, `strictPropertyInitialization: false` for Nest DI |
| `apps/web` | `esnext` | `bundler` | `jsx: react-jsx`, `noEmit: true`, `types: ["vite/client"]` — Vite owns the build |

## Vitest — one config, N projects

```ts
import react from '@vitejs/plugin-react';
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    projects: [
      { test: { name: 'contracts', root: './packages/contracts', environment: 'node', include: ['src/**/*.spec.ts'] } },
      { test: { name: 'db',        root: './packages/db',        environment: 'node', include: ['src/**/*.spec.ts'] } },
      {
        // esbuild cannot emit `design:paramtypes`, so Nest's constructor
        // injection breaks under the default transform. SWC emits it.
        plugins: [swc.vite({ module: { type: 'es6' } })],
        test: { name: 'api', root: './apps/api', environment: 'node', include: ['src/**/*.spec.ts'] },
      },
      {
        plugins: [react()],
        test: {
          name: 'web', root: './apps/web', environment: 'jsdom',
          setupFiles: ['./src/test/setup.ts'],
          include: ['src/**/*.spec.ts', 'src/**/*.spec.tsx'],
        },
      },
    ],
  },
});
```

Each Nx `test` target runs `vitest run --project <name>`; `pnpm test:watch` runs all of them. Tests live beside the code as `*.spec.ts(x)` with `globals: false` — import `describe`/`it`/`expect`/`vi` from `vitest`.

## Biome

One `biome.json` at the root. Key settings, and why:

- `vcs: { enabled: true, clientKind: "git", useIgnoreFile: true }` — `.gitignore` is the single ignore list.
- `files.includes` additionally excludes `**/dist`, `**/coverage`, `**/.nx`, `pnpm-lock.yaml`, and `packages/db/migrations` (generated SQL is not ours to format).
- `javascript.parser.unsafeParameterDecoratorsEnabled: true` — required for Nest's `@Inject()` parameter decorators.
- Override on `apps/api/**` turning `style/useImportType` off: Nest needs the runtime import for DI metadata, so a type-only import breaks injection.
- Override on spec files relaxing `suspicious/noConsole`.

### Code-shape rules

A second override on `**/*.ts` and `**/*.tsx` carries them: type-only imports first, then value imports grouped node → third-party → project and sorted alphabetically with blank lines between groups; 50 lines per function; 1000 lines per file; braces on every branch; 3 parameters before an options object; and two GritQL plugins, one capping if/else nesting at 2 and one keeping multi-property object literals off a single line. It has to be declared **before** the `apps/api/**` override, which switches `useImportType` back off for Nest DI.

Full config, per-rule examples and the plugin source: `references/lint-rules.md`.

## tRPC seam

- `packages/contracts` owns the Zod schemas. A shape used by both sides lives there and nowhere else.
- `apps/api/src/trpc/index.ts` exports the `AppRouter` **type** only.
- `apps/web` imports `#api-contract` for that type and builds the client with `@trpc/client` + `@tanstack/react-query`.

Adding an endpoint touches four files in this order: contract schema → Nest service → tRPC procedure → typed React call, each with its test.
