# Local infrastructure, git hooks and CI

## Two compose files

Split so a test run can never touch the development stack and leaves nothing behind. Different project `name:`, different ports, different storage.

### `compose.yml` — development

Data is **bind-mounted** under `./.data/<service>/`: visible, inspectable, throwaway, and gitignored. No top-level `volumes:` block — a named volume hides state where nobody looks and survives a `down`.

```yaml
name: <repo>

services:
  postgres:
    image: postgres:<exact>-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-app}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-app}
      POSTGRES_DB: ${POSTGRES_DB:-app}
    ports:
      - '${POSTGRES_PORT:-5432}:5432'
    volumes:
      - ./.data/postgres:/var/lib/postgresql   # NOT /data — see note below
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U ${POSTGRES_USER:-app} -d ${POSTGRES_DB:-app}']
      interval: 5s
      timeout: 5s
      retries: 20

  redis:
    image: redis:<exact>-alpine
    restart: unless-stopped
    command: ['redis-server', '--appendonly', 'yes']
    ports:
      - '${REDIS_PORT:-6379}:6379'
    volumes:
      - ./.data/redis:/data
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 5s
      timeout: 3s
      retries: 20
```

**Postgres data directory:** from Postgres 18 the image's `PGDATA` layout moved, and mounting `/var/lib/postgresql/data` leaves the bind mount empty while the container appears healthy. Mount `/var/lib/postgresql`. Check the image's own docs when you pin the version — this is the kind of thing that changes.

Every service carries a healthcheck so `docker compose up -d --wait` blocks until the stack is actually usable. Without `--wait`, the first migration races the database.

### `compose.test.yml` — ephemeral

```yaml
name: <repo>-test

services:
  postgres-test:
    image: postgres:<exact>-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-app}_test
    # Durability is worthless for a throwaway database; speed is not.
    command:
      - postgres
      - -c
      - fsync=off
      - -c
      - synchronous_commit=off
      - -c
      - full_page_writes=off
      - -c
      - max_connections=200
    ports:
      - '55432:5432'          # shifted, so it cannot collide with the dev stack
    tmpfs:
      - /var/lib/postgresql   # RAM only; nothing to clean up
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U ${POSTGRES_USER:-app} -d ${POSTGRES_DB:-app}_test']
      interval: 2s
      timeout: 3s
      retries: 30

  redis-test:
    image: redis:<exact>-alpine
    command: ['redis-server', '--save', '', '--appendonly', 'no']
    ports:
      - '56379:6379'
```

Tear it down with `-v` (`docker compose -f compose.test.yml down -v`).

## Environment

`.env` is gitignored; `.env.example` is committed and is the contract. A new variable lands in `.env.example` **in the same commit** that reads it.

`process.env` is read in exactly one file (`apps/api/src/env.ts`), validated there with Zod, and reached everywhere else through Nest's `ConfigService`.

Group the example file by owner so it stays readable:

```dotenv
# --- Runtime ---
NODE_ENV=development
API_PORT=3000
WEB_ORIGIN=http://localhost:4200

# --- Postgres (compose.yml) ---
POSTGRES_USER=app
POSTGRES_PASSWORD=app
POSTGRES_DB=app
POSTGRES_PORT=5432
DATABASE_URL=postgres://app:app@localhost:5432/app

# --- Redis / queues (compose.yml) ---
REDIS_PORT=6379
REDIS_URL=redis://localhost:6379

# --- Frontend ---
VITE_API_URL=http://localhost:3000
```

`.gitignore` must cover `.data/`, `.nx/`, `dist/`, `coverage/`, `node_modules/`, `*.tsbuildinfo`, and `.env*` with `!.env.example` re-included.

## Git hooks

Installed by husky (`"prepare": "husky"`). Both are affected-only, and both degrade gracefully when there is no base ref to diff against — otherwise the very first commit in a fresh repo fails.

**`.husky/pre-commit`** — cheap: format and lint the staged files, then verify only the projects those files belong to.

```sh
pnpm exec lint-staged

if git rev-parse --verify --quiet HEAD >/dev/null; then
  pnpm exec nx affected -t typecheck test --uncommitted --output-style=stream
else
  echo "No commit to diff against yet — verifying every project."
  pnpm exec nx run-many -t typecheck test --output-style=stream
fi
```

`--uncommitted` is the right selector here: at commit time the change is staged, not yet in a commit, so a branch-range diff would see nothing.

**`.husky/pre-push`** — the full sweep, still scoped to the branch.

```sh
base="$(git rev-parse --verify --quiet origin/main || git rev-parse --verify --quiet origin/master || true)"

if [ -n "$base" ]; then
  pnpm exec nx affected -t lint typecheck test build --base="$base" --head=HEAD --output-style=stream
else
  echo "No origin/main or origin/master to diff against — verifying every project."
  pnpm exec nx run-many -t lint typecheck test build --output-style=stream
fi
```

`lint-staged` runs `biome check --write --no-errors-on-unmatched` — one pass for both lint and format, and `--no-errors-on-unmatched` keeps a commit of only non-JS files from failing.

## GitHub Actions

Both workflows: on push and pull_request, `permissions: contents: read`, `actions/checkout@v7`, and a concurrency group that cancels superseded runs.

### `.github/workflows/ci.yml`

```yaml
name: CI
on: [push, pull_request]
permissions:
  contents: read
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  verify:
    name: Format · Lint · Types · Tests · Build
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0            # nx affected needs history
      - run: corepack enable        # BEFORE setup-node, or `cache: pnpm` finds no pnpm
      - uses: actions/setup-node@v7
        with:
          node-version-file: .nvmrc
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm format:check
      - run: pnpm lint
      - run: pnpm typecheck
      - run: pnpm test
      - run: pnpm build
```

Order matters: cheapest sensor first, so a formatting slip fails in seconds rather than after the build. CI runs the full `run-many` sweep — the affected scoping belongs in the hooks, where the developer is waiting.

### `.github/workflows/harness-score.yml`

```yaml
name: Harness Score
on: [push, pull_request]
permissions:
  contents: read
concurrency:
  group: harness-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  harness-score:
    name: Harness maturity (min L4)
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v7
      - uses: paladini/harness-score@v1
        with:
          min-level: '4'
```

Separate workflow, not a job in `ci.yml`: the harness score is about the repo's agent scaffolding, not its code, and it should stay readable as its own signal.
