# Codebase Summary

## Table of Contents

1. [Monorepo Layout Summary](#monorepo-layout-summary)
2. [Key Packages and Their Roles](#key-packages-and-their-roles)
3. [Build and Dependency Model](#build-and-dependency-model)
4. [Testing Approach](#testing-approach)
5. [CI/CD Overview](#cicd-overview)

---

## Monorepo Layout Summary

EverShop uses a **pnpm workspaces** monorepo structure with the following layout:

```
evershop/
├── packages/
│   ├── evershop/                    # Main platform package
│   ├── postgres-query-builder/      # SQL query builder library
│   └── create-evershop-app/         # Project scaffolding CLI
├── extensions/                      # Optional custom extensions
├── docs/                            # Project documentation
└── package.json                     # Root workspace config
```

### Root package.json

The root `package.json` defines workspaces for `packages/*` and `extensions/*`:

```json
{
  "workspaces": [
    "packages/*",
    "extensions/*"
  ]
}
```

---

## Key Packages and Their Roles

### 1. packages/evershop

**@evershop/evershop** - The main e-commerce platform

| Aspect | Details |
|--------|---------|
| Version | 2.1.1 |
| License | GPL-3.0 |
| Type | ESM (module) |
| Entry Point | `dist/bin/evershop.js` |

**Contents:**
- Core modules (auth, base, catalog, checkout, cms, cod, customer, graphql, oms, paypal, promotion, setting, stripe, tax)
- Express application and HTTP server
- React admin and storefront interfaces
- GraphQL schema and resolvers
- CLI commands (build, dev, start, install, theme, user, seed)

**Exports Map:**
- `lib/helpers` - Path constants and utilities
- `lib/postgres` - Database connection
- `lib/router` - Route management
- `lib/event` - Event emission and subscribers
- `lib/widget` - Widget system
- `lib/cronjob` - Scheduled jobs
- `components/common/*`, `components/admin/*`, `components/frontStore/*` - React components
- `*/services` - Domain services for graphql, catalog, customer, setting, checkout, oms, cms

### 2. packages/postgres-query-builder

**@evershop/postgres-query-builder** - PostgreSQL query builder

| Aspect | Details |
|--------|---------|
| Version | 2.0.1 |
| License | MIT |
| Type | CommonJS |
| Entry Point | `dist/index.js` |

A lightweight SQL query builder for PostgreSQL that provides a fluent API for building queries. Used throughout the platform for database operations.

### 3. packages/create-evershop-app

**create-evershop-app** - Project scaffolding CLI

A command-line tool to scaffold new EverShop projects with a pre-configured setup.

### 4. extensions/

Optional workspace packages for custom extensions. The root `package.json` includes `extensions/*` in workspaces, but this folder may be empty in the base installation.

---

## Build and Dependency Model

### Build System Overview

```
Source (src/) → SWC Compile → Distribution (dist/)
```

### Compilation Commands

| Command | Purpose |
|---------|---------|
| `npm run compile` | SWC: `packages/evershop/src` → `packages/evershop/dist` |
| `npm run compile:db` | SWC: `packages/postgres-query-builder/src` → `dist` |
| `npm run compile:tsc` | TypeScript: optional full compile for publishing |
| `npm run build` | Webpack: builds admin/storefront bundles (requires compile first) |
| `npm run build-fast` | Build with `--skip-minify` for faster builds |

### Build Model Details

1. **Day-to-day Development**: SWC (Super-fast Web Compiler)
   - Fast transpilation of TypeScript/JavaScript
   - Preserves directory structure
   - Copies non-code files (GraphQL, JSON, SCSS, CSS)

2. **NPM Publishing**: TypeScript (`compile:tsc`)
   - Full type checking
   - Asset copying via `copyfiles`
   - Triggered by `prepack` script

3. **Runtime**: Loads from `dist/`
   - Module bootstrap loader looks for `bootstrap.js` in each module folder
   - Compiled from `.ts` if applicable

### Dependency Model

**Internal Dependencies:**
- `@evershop/evershop` depends on `@evershop/postgres-query-builder`
- Extensions depend on `@evershop/evershop` for APIs and components

**External Dependencies (key ones):**
- Express.js - Web framework
- React 17.x - UI library
- GraphQL - API layer
- pg - PostgreSQL driver
- node-config - Configuration management
- Webpack - Bundling
- Winston - Logging

---

## Testing Approach

### Test Framework

**Jest** is the primary testing framework with the following configuration:

```javascript
// jest.config.js
{
  testEnvironment: "node",
  testMatch: ["**/dist/**/tests/**/unit/**/*.test.[jt]s"],
  modulePathIgnorePatterns: ["<rootDir>/packages/evershop/src/"]
}
```

### Key Testing Principles

1. **Tests Run Against Compiled Code**
   - Jest only matches files in `dist/` directory
   - Source tests must be in `packages/evershop/src/**/tests/**`
   - After compilation, they end up in `dist/.../tests/.../unit/...`

2. **Always Compile Before Testing**
   - Run `npm run compile` (and `npm run compile:db` if needed) before `npm run test`
   - Otherwise tests will be missing or stale

3. **Module Name Mapping**
   - `@evershop/postgres-query-builder` maps to `packages/postgres-query-builder/dist`

4. **Test Environment Variables**
   - `ALLOW_CONFIG_MUTATIONS=true` required for tests
   - `NODE_OPTIONS=--experimental-vm-modules` for ESM support

### Running Tests

```bash
# Full test command (as defined in package.json)
npm run test

# Which executes:
ALLOW_CONFIG_MUTATIONS=true NODE_OPTIONS=--experimental-vm-modules node_modules/jest/bin/jest.js
```

### Test Organization

- **Unit Tests**: Located in `**/tests/unit/` within modules
- **Integration Tests**: Located in `tests/intergration/` (note: typo in original) - not matched by default Jest config

---

## CI/CD Overview

### GitHub Actions Workflow

**File**: `.github/workflows/build_test.yml`

**Trigger**: Pull requests

**Matrix Testing**: Node.js versions 20 and 22

**Workflow Steps**:

```yaml
1. Checkout code
2. Setup Node.js (version from matrix)
3. Upgrade npm to version 9
4. Install dependencies: npm install
5. Compile platform: npm run compile
6. Compile query builder: npm run compile:db
7. Run tests: npm run test
```

### CI Requirements

| Step | Command | Purpose |
|------|---------|---------|
| Install | `npm install` | Install all dependencies |
| Compile | `npm run compile` | Build evershop package |
| Compile DB | `npm run compile:db` | Build query builder |
| Test | `npm run test` | Run Jest test suite |

### What is NOT in CI

- **Lint**: `npm run lint` is NOT part of CI pipeline
- Must be run manually by developers

### Best Practices for CI

1. Always ensure tests pass before opening PR
2. Run lint locally: `npm run lint`
3. Test on both Node 20 and 22 if possible
4. Include tests for new features and bug fixes

---

## Summary

The EverShop monorepo is well-organized with clear separation between the main platform, the query builder library, and the scaffolding tool. The build system uses SWC for fast development builds and TypeScript for production publishing. Testing is Jest-based and runs against compiled code, requiring proper compilation before test execution. The CI pipeline is straightforward, testing on Node 20 and 22, but relies on developers to run linting manually before creating pull requests.