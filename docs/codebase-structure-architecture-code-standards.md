# Codebase Structure, Architecture, and Code Standards

## Table of Contents

1. [Detailed Directory Structure](#detailed-directory-structure)
2. [Runtime Architecture](#runtime-architecture)
3. [Events and Subscribers Pattern](#events-and-subscribers-pattern)
4. [Extensions Model](#extensions-model)
5. [Configuration System](#configuration-system)
6. [Linting and Code Style Standards](#linting-and-code-style-standards)
7. [Git/PR Conventions](#gitpr-conventions)

---

## Detailed Directory Structure

### Root Level

```
evershop/
├── packages/
│   ├── evershop/                    # Main platform
│   ├── postgres-query-builder/      # Query builder
│   └── create-evershop-app/         # Project scaffold
├── extensions/                      # Custom extensions
├── docs/                            # Documentation
├── .github/workflows/               # CI configuration
├── package.json                     # Root workspace
├── jest.config.js                  # Test configuration
├── eslint.config.js                 # Linting config
└── LICENSE                          # GPL-3.0
```

### packages/evershop Structure

```
packages/evershop/
├── src/
│   ├── bin/                         # CLI and startup
│   │   ├── evershop.js              # Main CLI entry
│   │   ├── dev/                     # Development server
│   │   ├── start/                   # Production server
│   │   ├── build/                   # Build process
│   │   ├── install/                 # Installation/migration
│   │   ├── extension/               # Extension loader
│   │   ├── lib/                     # Core libraries
│   │   │   ├── router/              # Route management
│   │   │   ├── event/               # Event system
│   │   │   ├── postgres/            # Database
│   │   │   ├── log/                 # Logging
│   │   │   ├── util/                # Utilities
│   │   │   ├── webpack/             # Build tooling
│   │   │   ├── cronjob/             # Scheduled jobs
│   │   │   └── mail/                # Email helpers
│   │   └── lib/                     # Bootstrap and startup
│   │       ├── app.js               # Express app creation
│   │       ├── loadModules.js       # Module loading
│   │       ├── startUp.js           # Startup sequence
│   │       └── bootstrap/           # Module bootstrap
│   ├── modules/                     # Core modules
│   │   ├── auth/                    # Authentication
│   │   ├── base/                    # Base/infrastructure
│   │   ├── catalog/                 # Products/categories
│   │   ├── checkout/                # Cart/checkout
│   │   ├── cms/                     # Pages/widgets
│   │   ├── cod/                     # Cash on delivery
│   │   ├── customer/                # Customer accounts
│   │   ├── graphql/                 # GraphQL infrastructure
│   │   ├── oms/                     # Order management
│   │   ├── paypal/                  # PayPal integration
│   │   ├── promotion/               # Coupons/discounts
│   │   ├── setting/                 # Store settings
│   │   ├── stripe/                  # Stripe integration
│   │   └── tax/                     # Tax calculation
│   ├── components/                  # Shared React components
│   │   ├── common/                  # Common components
│   │   ├── admin/                   # Admin UI components
│   │   └── frontStore/              # Storefront components
│   └── types/                       # TypeScript type definitions
├── dist/                            # Compiled output
├── package.json                     # Package config
└── .swcrc                           # SWC configuration
```

### Module Directory Pattern

Each core module follows a consistent structure:

```
modules/<module-name>/
├── bootstrap.js                     # Startup initialization
├── graphql/
│   ├── types/                       # GraphQL schema files
│   │   ├── *.graphql               # Shared types
│   │   ├── *.admin.graphql         # Admin-only types
│   │   └── *.resolvers.js          # Resolver implementations
├── api/
│   └── <handler>/
│       ├── route.json               # Route definition
│       ├── route.js                 # Route handler
│       └── payloadSchema.json       # Request validation
├── pages/
│   ├── admin/                       # Admin pages
│   │   └── <page>/
│   │       ├── route.json
│   │       └── index.js            # React component
│   └── frontStore/                  # Storefront pages
│       └── <page>/
│           ├── route.json
│           └── index.js
├── migration/                        # Database migrations
├── services/                        # Domain logic
└── subscribers/
    └── <event-name>/               # Event handlers
        └── *.js
```

---

## Runtime Architecture

### Express App Assembly

The Express application is created in `packages/evershop/src/bin/lib/app.js` via the `createApp()` function:

1. **Instantiate Express** - Creates Express app with `trust proxy` enabled
2. **Load Core Modules** - Iterates through all core modules (defined in `loadModules.js`)
3. **Load Extensions** - Loads each enabled extension from `system.extensions`
4. **Register Middleware** - Applies default middleware stack
5. **Register Routes** - Maps all routes with their handlers
6. **Catch-all Handler** - Final middleware for unmatched routes

### Core Module List

The following 14 core modules are explicitly loaded (from `loadModules.js`):

| Module | Purpose |
|--------|---------|
| auth | Sessions, cookies, admin vs storefront auth |
| base | Shared infrastructure, base migrations |
| catalog | Products, categories, attributes |
| checkout | Cart and checkout flow |
| cms | Pages, widgets, SEO, URL rewrites |
| cod | Cash on delivery payment |
| customer | Customer accounts and addresses |
| graphql | GraphQL schema and HTTP endpoints |
| oms | Order management system |
| paypal | PayPal checkout integration |
| promotion | Coupons and promotions |
| setting | Shop settings management |
| stripe | Stripe payment integration |
| tax | Tax calculation and configuration |

### Startup Sequence

The startup process (`packages/evershop/src/bin/lib/startUp.js`) follows this order:

1. **Create Express App** - `createApp()`
2. **Create HTTP Server** - `http.createServer(app)`
3. **Load Module Bootstrap Scripts** - Execute each module's `bootstrap.js`
4. **Lock Hooks and Registries** - `lockHooks()`, `lockRegistry()`
5. **Validate Configuration** - `validateConfiguration(config)`
6. **Run Migrations** - Execute pending database migrations
7. **Start Listening** - Bind to port and begin accepting requests
8. **Spawn Child Processes**:
   - Event subscriber process (`event-manager`)
   - Cron job process

### GraphQL Schema Building

GraphQL schema is built dynamically from module files:

- **`buildTypeDefs`** (`modules/graphql/services/buildTypes.js`):
  - Globs `MODULESPATH/*/graphql/types/**/*.graphql`
  - Also includes extension GraphQL types
  - Merges with `@graphql-tools/merge`
  - **Storefront** build ignores `*.admin.graphql` files
  - **Admin** build includes all GraphQL files

- **`buildResolvers`**:
  - Globs `**/*.resolvers.{js,ts}` files
  - Different `ignoredExtensions` for admin vs storefront
  - Non-admin loading ignores `.ts` files in merge step

### Route Discovery

Routes are discovered and loaded via `loadModuleRoutes` (`packages/evershop/src/lib/router/loadModuleRoutes.js`):

- Scans `pages/admin/`, `pages/frontStore/`, and `api/` directories
- Reads `route.json` for each route definition
- Registers Express handlers for each route

---

## Events and Subscribers Pattern

### Event System Overview

EverShop uses an event-driven architecture for asynchronous processing:

1. **Event Emission** - Code calls `emit(eventName, data)`
2. **Event Storage** - Events are persisted to the `event` table in PostgreSQL
3. **Event Processing** - A dedicated subscriber process reads and processes events

### Event Emission

```typescript
// packages/evershop/src/lib/event/emitter.ts
import { emit } from '@evershop/evershop/lib/event';

// Usage
await emit('orderCreated', { orderId: '123', customerId: '456' });
```

### Event Types

Events are typed via **`EventDataRegistry`** in `packages/evershop/src/types/event.ts`. This registry documents all available events with their data structures.

### Subscriber Implementation

Subscribers are placed in module directories under `subscribers/<eventName>/`:

```
modules/catalog/subscribers/productCreated/
└── index.js  // Default export is the handler
```

The subscriber loader (`packages/evershop/src/lib/event/loadSubscribers.ts`):
- Scans `subscribers/<eventDir>/*.js` files
- Only loads compiled `.js` files (not `.ts`)
- Each file's default export is the handler function

### Event Processing Pipeline

1. **EventStorage** - Reads events from the `event` table
2. **EventProcessor** - Processes each event by calling registered subscribers
3. **Notification** - Uses PostgreSQL `LISTEN new_event` with polling fallback

---

## Extensions Model

### Extension Configuration

Extensions are configured in `config/default.json` under `system.extensions`:

```json
{
  "system": {
    "extensions": [
      {
        "name": "myExtension",
        "path": "./extensions/my-extension",
        "enabled": true,
        "priority": 10
      }
    ]
  }
}
```

### Extension Loading

The extension loader (`packages/evershop/src/bin/extension/index.ts`):
- Reads `system.extensions` from config
- Validates extension names (must not collide with core modules)
- Sorts by `priority` (lower = loads earlier)
- Logs warnings for invalid/missing paths

### Extension Requirements

| Environment | Requirement |
|-------------|-------------|
| Production (node_modules) | Must have `dist/` directory |
| Development (local path) | Must have `src/` directory |
| Both | Must have `enabled: true` in config |

### Extension Structure

Extensions follow the same directory pattern as core modules:
- `bootstrap.js` - Startup initialization
- `graphql/types/` - GraphQL schema
- `api/` - REST API handlers
- `pages/admin/` - Admin pages
- `pages/frontStore/` - Storefront pages
- `services/` - Domain logic
- `subscribers/` - Event handlers

---

## Configuration System

### Configuration Approach

EverShop uses the **`node-config`** package for configuration management:

- Configuration files in `config/` directory
- Default config: `config/default.json`
- Environment-specific overrides supported

### Configuration Access

```typescript
// packages/evershop/src/lib/util/getConfig.ts
import { getConfig } from '@evershop/evershop/lib/util';

// Usage
const shopName = getConfig('shop.name', 'My Shop');
const dbConfig = getConfig('system.database');
```

### Common Configuration Paths

| Path | Description |
|------|-------------|
| `shop.*` | Shop metadata (name, email, etc.) |
| `system.*` | System config (database, session, extensions) |
| `catalog.*` | Catalog settings |
| `checkout.*` | Checkout configuration |
| `pricing.*` | Pricing rules |
| `themeConfig.*` | Theme settings |
| `oms.*` | Order management settings |

### Configuration Mutations

The `ALLOW_CONFIG_MUTATIONS` environment variable controls runtime config changes:

- **`true`**: Allowed during dev, test, and start initialization
- **`false`**: Locked after bootstrap completes

Set in:
- Root `test` script
- `dev` spawn
- `start` env init

---

## Linting and Code Style Standards

### ESLint Configuration

EverShop uses **ESLint 9** with a flat config configuration (`eslint.config.js`).

### Key Rules

| Rule | Setting | Notes |
|------|---------|-------|
| `no-console` | **error** | Use logger instead |
| `prefer-const` | error | Prefer const over let |
| `import/order` | warn | Sort imports alphabetically |
| `jsx-a11y/*` | varies | Accessibility rules |
| `react/*` | varies | React best practices |

### Ignored Paths

The following directories are excluded from linting:

```
/node_modules/
**/*test.js
**/tests/**
**/create-evershop-app/**
**/.evershop/**
/.vscode/
/.git/
/.idea/
**/extensions/**
**/public/**
**/themes/**
**/media/**
**/dist/**
**/packages/*/dist/**
**/packages/evershop/dist/**
**/packages/postgres-query-builder/dist/**
**/packages/product_review/**
**/packages/resend/**
```

### Running Lint

```bash
# Run lint with auto-fix
npm run lint

# Which executes:
eslint --fix --ext .js,.jsx,.ts,.tsx ./packages
```

### Code Style Notes

1. **No console.log** - Use Winston logger from `packages/evershop/src/lib/log/logger.js`
2. **Prefer const** - Use `const` over `let` wherever possible
3. **Import ordering** - Grouped and alphabetized (warn level)
4. **TypeScript** - Full support with `@typescript-eslint`
5. **React** - JSX and hooks supported with react plugin

---

## Git/PR Conventions

### Repository Branches

| Branch | Purpose |
|--------|---------|
| `dev` | Target branch for contributions |
| `main` | Stable releases (if applicable) |

### Contribution Workflow

1. **Fork** the repository
2. **Clone** to local machine
3. **Create** a new feature branch
4. **Make** changes with tests
5. **Run** lint and tests locally
6. **Push** to your fork
7. **Open** PR against `dev` branch

### PR Requirements

- **Tests**: Include tests for new features and bug fixes
- **Description**: Clear description of changes
- **Target**: PRs should target the `dev` branch

### PR Template

See `.github/pull_request_template.md` for the required PR template.

### Commit Messages

Follow conventional commit format (not strictly enforced but recommended):

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

---

## Summary

EverShop's architecture is built on a modular foundation with clear separation between core modules, extensions, and the runtime environment. The Express-based application loads modules dynamically, builds GraphQL schemas from filesystem conventions, and uses an event-driven system for asynchronous processing. Configuration is managed through node-config with environment-specific overrides. Code quality is enforced through ESLint with strict rules around console usage and TypeScript/React best practices. Contributions follow a straightforward GitHub flow targeting the `dev` branch with test requirements for all changes.