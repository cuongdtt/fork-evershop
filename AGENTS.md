# AGENTS.md — AI & contributor guide (EverShop monorepo)

This file is the primary onboarding document for **AI coding agents** and developers working in this repository. It summarizes layout, commands, runtime behavior, and conventions so changes stay aligned with the framework.

## Product snapshot

EverShop is a **GPL-3.0** e-commerce platform: **Node.js**, **Express**, **PostgreSQL**, **GraphQL**, and **React** (17.x in core `package.json`). User-facing install roots use **`config`** (the [`config`](https://github.com/node-config/node-config) package), typically via `config/default.json` at the project root. Library path resolution uses **`process.cwd()`** when `@evershop/evershop` runs from `node_modules`.

Official docs: https://evershop.io/docs/development/getting-started/introduction

## Monorepo layout

| Path | Role |
|------|------|
| `packages/evershop/` | Platform: CLI (`evershop`), Express app, admin + storefront React, GraphQL, all core **modules** |
| `packages/postgres-query-builder/` | `@evershop/postgres-query-builder` — PostgreSQL query builder (**MIT**), build output in `dist/` |
| `packages/create-evershop-app/` | CLI to scaffold new EverShop projects (`create-evershop-app`) |
| `extensions/` | Optional workspace packages for custom extensions (`package.json` lists `extensions/*`; folder may be empty) |

Root `package.json` uses **workspaces**: `packages/*`, `extensions/*`.

## Commands (use `pnpm` where applicable)

| Script | What it does |
|--------|----------------|
| `compile` | SWC: `packages/evershop/src` → `packages/evershop/dist` |
| `compile:db` | SWC: `packages/postgres-query-builder/src` → `dist` |
| `build` | Runs `dist/bin/build` (needs **`compile` first**) |
| `dev` | Dev server; spawns child with **`ALLOW_CONFIG_MUTATIONS=true`** |
| `start` | Production server (`NODE_ENV=production` via start env) |
| `setup` | `evershop install` — DB migrations / schema |
| `test` | Jest (root `jest.config.js`); **`ALLOW_CONFIG_MUTATIONS=true`**, `NODE_OPTIONS=--experimental-vm-modules` |
| `lint` | ESLint with `--fix` on `./packages` |
| `build-fast` | Build with `--skip-minify` |

**CI** (`.github/workflows/build_test.yml`): PRs; Node **20** and **22**; `npm install`, `compile`, `compile:db`, `test`. **Lint is not in CI.**

**Typical loop:** install deps → `compile` + `compile:db` → configure Postgres + `config` → `setup` once → `dev` or `test` / `lint`.

## Build model

- Day-to-day: **SWC** (`compile`), not root `tsc`.
- **`compile:tsc`**: optional TypeScript compile path for `packages/evershop`.
- **`@evershop/evershop` `prepack`**: `tsc` + asset copy for npm publish.
- Runtime loads **`dist/`**. Module bootstrap loader only looks for **`bootstrap.js`** in each module folder (compiled from `.ts` if applicable).

## Testing

- **Jest** config at repo root; environments **`node`**, ESM via `NODE_OPTIONS=--experimental-vm-modules`.
- **`testMatch`**: `**/dist/**/tests/**/unit/**/*.test.[jt]s` — tests execute from **compiled** trees only; `modulePathIgnorePatterns` excludes `packages/evershop/src/`.
- **Always run `compile` (and `compile:db` if needed) before `test`**, or tests will be missing/stale.
- **Name mapper**: `@evershop/postgres-query-builder` → `packages/postgres-query-builder/dist`.
- Source tests often live under `packages/evershop/src/**/tests/**` as `.test.js`; they must end up under `dist/.../tests/.../unit/...` for the default Jest run.
- Folders like `tests/intergration/` (sic) under modules are **not** matched by the default `unit` glob — integration suites may require a separate Jest config or command unless moved/renamed.

## Linting (`eslint.config.js`)

- ESLint **9** flat config: TypeScript parser/plugin, React + **jsx-a11y** recommended.
- **`no-console` is `error`** — use `packages/evershop/src/lib/log/logger.js` patterns instead.
- Many trees are **ignored** (`dist`, `extensions/**`, `themes/**`, `public/**`, `media/**`, `**/tests/**`, `create-evershop-app/**`, etc.). Read the full `ignores` array before assuming a path is linted.

## Runtime architecture

### Express app assembly

`packages/evershop/src/bin/lib/app.js` **`createApp()`**:

1. Instantiates Express (`trust proxy` enabled).
2. For each **core module** then each **enabled extension**: loads **middleware** definitions and **routes** from that path.
3. Applies default middleware, registers Express methods per route, ends with catch-all `Handler.middleware()`.

Core module list is **explicit** in `packages/evershop/src/bin/lib/loadModules.js` (`getCoreModules()`): `auth`, `base`, `catalog`, `checkout`, `cms`, `cod`, `customer`, `graphql`, `oms`, `paypal`, `promotion`, `setting`, `stripe`, `tax`.

### Module directories

Core and extensions share the same layout patterns under `.../modules/<name>/` (core) or extension `dist/` (or `src` in dev for some tooling):

| Area | Purpose |
|------|--------|
| `bootstrap.js` | Default export invoked at startup; register services, filters, etc. |
| `graphql/types/` | `*.graphql`, `*.resolvers.js` (and `.admin.*` variants) |
| `api/<handler>/` | `route.json`, handlers, optional `payloadSchema.json`, bracket middleware files |
| `pages/admin/`, `pages/frontStore/` | `route.json` + React entry/components |
| `migration/` | DB migrations |
| `services/` | Domain logic |
| `subscribers/<eventName>/` | Async handlers for events (see below) |

### GraphQL schema

- **`buildTypeDefs`** (`modules/graphql/services/buildTypes.js`): globs `MODULESPATH/*/graphql/types/**/*.graphql` plus each extension’s `graphql/types`; merges with `@graphql-tools/merge`. **Storefront** build **ignores** `*.admin.graphql`; **admin** includes them.
- **`buildResolvers`**: globs `**/*.resolvers.{js,ts}` with different `ignoredExtensions` for admin vs storefront; non-admin loading ignores `.ts` in the merge step — **rely on compiled `.js` in production paths**. Development can dynamic-import with cache-busting.

### Events

- **`emit()`** (`lib/event/emitter.ts`): persists to the **`event`** table via `@evershop/postgres-query-builder`, then the event worker/subscriber pipeline processes rows (see `EventProcessor` and subscriber loading).
- **Typed events**: extend **`EventDataRegistry`** in `packages/evershop/src/types/event.ts` (documented in-file with examples).
- **Subscribers**: `lib/event/loadSubscribers.ts` scans **`subscribers/<eventDir>/*.js`** only (compiled `.js` under each module extension path). Default export per file is the handler.

### Extensions

`packages/evershop/src/bin/extension/index.ts` reads **`system.extensions`** from config via `getConfig('system.extensions', [])`.

- **`name`** must not collide with a core module or another extension.
- **`enabled: true`** to load; invalid/missing paths log warnings or exit depending on mode.
- **Production** (or `resolve` under `node_modules`): **`dist`** must exist.
- **Development** (local path, not `node_modules`): **`src`** must exist; `path` for loading uses `dist` under that package — ensure build output is present for route/graphql scans.
- Sorted by **`priority`** (lower = earlier).

### Configuration

- **`getConfig(path, default)`** (`lib/util/getConfig.ts`): typed wrapper over `config`; paths include `shop.*`, `system.*` (database, session, extensions, stripe, paypal, …), `catalog.*`, `checkout.*`, `pricing.*`, `themeConfig.*`, `oms.*`, etc.
- **`ALLOW_CONFIG_MUTATIONS`**: set **`true`** in root **`test`** script, **`dev`** spawn, and **`start` env init** so runtime config updates are permitted where the framework expects them.

### Important paths (`lib/helpers.ts` → `CONSTANTS`)

`ROOTPATH`, `LIBPATH`, `MODULESPATH`, `PUBLICPATH`, `MEDIAPATH`, `NODEMODULEPATH`, `THEMEPATH`, `CACHEPATH` (`.evershop`), `BUILDPATH` (`.evershop/build`), `ADMIN_COLLECTION_SIZE`.

## Published package (`@evershop/evershop`)

`packages/evershop/package.json` **`exports`** map documents supported entry points: `lib/helpers`, `lib/postgres`, `lib/router`, `lib/event`, `lib/widget`, `lib/cronjob`, `lib/middleware/delegate`, `components/common|admin|frontStore/*`, and `*/services` for graphql, catalog, customer, setting, checkout, oms, cms. Prefer these for extensions rather than deep internal imports.

## Other tooling

- **Cypress** is a root **devDependency** but there is **no checked-in `cypress.config`** in this fork — E2E may be optional or maintained elsewhere.
- **Husky**: root `prepare` runs `husky install` for git hooks.

## Git & PRs

- Target branch for contributions: **`dev`** (`CONTRIBUTING.md`).
- PR template expects tests for fixes/features: `.github/pull_request_template.md`.

## Agent checklist (quick reference)

1. Touch the **smallest surface**: one module under `packages/evershop/src/modules/<name>/` or shared `lib/`.
2. **Compile before test** or running CLI from `dist`.
3. GraphQL: add **`*.graphql`** + resolvers; use **`.admin.graphql`** only for admin-only schema.
4. Background work: consider **`emit`** + **`subscribers/<event>/*.js`** (after compile).
5. New HTTP APIs: **`api/.../route.json`** + middleware naming conventions already used in sibling modules.
6. Do not **name an extension** like a **core module**.
7. Respect **ESLint ignore** lists; **`no-console`** failures are common if you log ad hoc.
8. For product behavior, API lists, and theme authoring, cross-check **evershop.io** docs.

## Licenses

- **`packages/evershop`**: **GPL-3.0**
- **`packages/postgres-query-builder`**: **MIT**

See `LICENSE`, `CONTRIBUTING.md`, and `CODE_OF_CONDUCT.md`.
