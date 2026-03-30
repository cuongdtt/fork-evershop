# EverShop Learning Guide - Step by Step Deep Dive

This guide will walk you through the EverShop e-commerce platform architecture systematically. Each step builds upon the previous one.

## Prerequisites
- Basic knowledge of Node.js, Express, React, and PostgreSQL
- The project cloned and dependencies installed (`npm install` or `pnpm install`)

---

## Step 1: Project Overview & High-Level Architecture

**Goal**: Understand what EverShop is and its overall structure.

### 1.1 Read the Architecture Document
Start by reading the main architecture overview:
```
docs/ARCHITECTURE.md
```

This will give you the big picture of:
- Monorepo structure
- Core modules and extensions
- Build system (SWC compilation)

### 1.2 Explore the Directory Structure
Navigate to these key directories:
```bash
# Core platform code
packages/evershop/src/

# Core modules (where most business logic lives)
packages/evershop/src/modules/

# Shared libraries
packages/evershop/src/lib/

# PostgreSQL query builder (separate MIT-licensed package)
packages/postgres-query-builder/
```

### 1.3 Key Configuration Files
Look at:
- `package.json` - workspace configuration
- `config/default.json` - main configuration (if exists in your setup)
- `packages/evershop/package.json` - core package exports

**Checkpoint**: Can you list the 13 core modules? (Hint: look at AGENTS.md line about `getCoreModules()`)

---

## Step 2: The Module System - Heart of EverShop

**Goal**: Understand how EverShop organizes code into modules.

### 2.1 Module Directory Structure
Each module follows a consistent pattern. Let's examine one core module:

```bash
# Pick any core module, for example:
packages/evershop/src/modules/catalog/
```

Within each module, you'll find:
```
modules/<name>/
├── bootstrap.js          # Module initialization
├── graphql/              # GraphQL schema and resolvers
│   └── types/
├── api/                  # REST API endpoints
├── pages/                # React pages (admin/ frontStore)
├── services/             # Business logic
├── migration/            # Database migrations
└── subscribers/          # Event handlers
```

### 2.2 Hands-on: Explore the Catalog Module
1. Navigate to `packages/evershop/src/modules/catalog/`
2. Look at the `bootstrap.js` - what does it export?
3. Check `graphql/types/` - what GraphQL types are defined?
4. Look at `api/` - what API endpoints exist?

### 2.3 Module Loading Mechanism
Read how modules are loaded:
```
packages/evershop/src/bin/lib/loadModules.js
```

This shows how core modules and extensions are discovered and loaded at startup.

**Checkpoint**: What function returns the list of core modules?

---

## Step 3: Application Bootstrap & Express Setup

**Goal**: Understand how the Express app is created and configured.

### 3.1 Application Entry Point
The main entry point is:
```
packages/evershop/src/bin/lib/app.js
```

This file's `createApp()` function:
1. Creates Express instance with `trust proxy`
2. Loads all modules (core + extensions)
3. Applies middleware from each module
4. Registers routes
5. Sets up error handling

### 3.2 The Bootstrap Process
Read:
```
packages/evershop/src/bin/lib/loadModules.js
```

Pay attention to:
- How modules are sorted by priority
- How `bootstrap.js` from each module is called
- The difference between core modules and extensions

### 3.3 Extension System
Extensions work the same way as core modules. Read:
```
packages/evershop/src/bin/extension/index.ts
```

**Key Concept**: Extensions allow you to add/override functionality without modifying core code.

**Checkpoint**: What configuration key controls which extensions are enabled?

---

## Step 4: GraphQL Architecture

**Goal**: Understand how GraphQL is implemented in EverShop.

### 4.1 Type Definitions
GraphQL types are defined in:
```
packages/evershop/src/modules/graphql/services/buildTypes.js
```

This file:
- Globs all `*.graphql` files from modules
- Merges them using `@graphql-tools/merge`
- Separates admin vs storefront schemas (`.admin.graphql` files)

### 4.2 Resolvers
Resolvers are loaded by:
```
packages/evershop/src/modules/graphql/services/buildResolvers.js
```

Look at a specific example:
```
packages/evershop/src/modules/catalog/graphql/types/
```

You'll see pairs like:
- `Product.graphql` - type definitions
- `Product.resolvers.js` - resolver implementation

### 4.3 GraphQL Endpoint
The GraphQL endpoint is set up in the `graphql` module. Look at:
```
packages/evershop/src/modules/graphql/
```

**Checkpoint**: How does EverShop distinguish between admin-only and storefront GraphQL schemas?

---

## Step 5: Database Layer & Query Builder

**Goal**: Understand database interactions.

### 5.1 PostgreSQL Query Builder
This is a separate MIT-licensed package:
```
packages/postgres-query-builder/
```

Read:
- `packages/postgres-query-builder/src/index.ts` - main exports
- `packages/postgres-query-builder/src/Connection.ts` - database connection handling

### 5.2 Using the Query Builder in Modules
Look at how modules use it:
```
packages/evershop/src/modules/catalog/services/
```

Search for imports of `@evershop/postgres-query-builder` to see usage patterns.

### 5.3 Migrations
Database schema changes are handled via migrations:
```
packages/evershop/src/modules/catalog/migration/
```

Each module can have its own migrations that run during `evershop install`.

**Checkpoint**: What command runs database setup/migrations?

---

## Step 6: API & Routing System

**Goal**: Understand REST API structure.

### 6.1 Route Definition
API routes are defined in:
```
packages/evershop/src/modules/<module>/api/<route>/
```

Each route has:
- `route.json` - route configuration (method, path, etc.)
- Handler files (e.g., `[id].js` for dynamic routes)
- Optional `payloadSchema.json` for validation
- Middleware files in brackets like `[auth].js`, `[validation].js`

### 6.2 Example Route
Look at:
```
packages/evershop/src/modules/catalog/api/products/
```

This shows:
- How routes are structured
- How middleware is applied
- How handlers process requests

### 6.3 Middleware System
Middleware can be:
- Defined in `[name].js` files within route folders
- Registered globally in `bootstrap.js`

Look at middleware examples:
```
packages/evershop/src/modules/auth/api/middleware/
```

**Checkpoint**: What file defines the route configuration (method, path) for an API endpoint?

---

## Step 7: Frontend - React Pages

**Goal**: Understand the React frontend architecture.

### 7.1 Page Structure
Pages are in:
```
packages/evershop/src/modules/<module>/pages/
├── admin/          # Admin panel pages
└── frontStore/     # Customer-facing pages
```

### 7.2 Page Components
Each page has:
- `route.json` - route configuration
- React components (`.jsx` files)
- GraphQL queries for data fetching

### 7.3 Example: Product Page
Look at:
```
packages/evershop/src/modules/catalog/pages/frontStore/productView/
```

You'll see:
- How pages are structured
- How they fetch data via GraphQL
- How they use shared components

### 7.4 Shared Components
Common UI components are in:
```
packages/evershop/src/components/
```

Organized by:
- `common/` - Used everywhere
- `admin/` - Admin-specific
- `frontStore/` - Storefront-specific

**Checkpoint**: Where are admin panel pages located vs storefront pages?

---

## Step 8: Services & Business Logic

**Goal**: Understand how business logic is organized.

### 8.1 Services Pattern
Business logic lives in:
```
packages/evershop/src/modules/<module>/services/
```

Services are plain JavaScript/TypeScript functions that:
- Encapsulate business rules
- Interact with the database
- Can be imported by API handlers, GraphQL resolvers, or other services

### 8.2 Example Service
Look at:
```
packages/evershop/src/modules/catalog/services/product/
```

Notice how services:
- Are organized by domain entity
- Export functions like `createProduct()`, `updateProduct()`, etc.
- Use the query builder for database operations

### 8.3 Service Exports
Important services are exported in `package.json` for extensions to use:
```json
"exports": {
  "./catalog/services": "./dist/modules/catalog/services/index.js",
  ...
}
```

**Checkpoint**: Why are services exported from package.json?

---

## Step 9: Event System

**Goal**: Understand the event-driven architecture.

### 9.1 Event Emitter
Events are emitted using:
```
packages/evershop/src/lib/event/emitter.ts
```

The `emit()` function:
- Persists events to the `event` table
- Triggers async processing

### 9.2 Event Types
Typed events are defined in:
```
packages/evershop/src/types/event.ts
```

### 9.3 Subscribers (Event Handlers)
Subscribers are in:
```
packages/evershop/src/modules/<module>/subscribers/<eventName>/
```

Example: Look for any `subscribers/` folder in a module.

Each subscriber file:
- Is a `.js` file
- Exports a default function as the handler
- Runs asynchronously when the event is emitted

### 9.4 Loading Subscribers
Subscribers are loaded by:
```
packages/evershop/src/lib/event/loadSubscribers.ts
```

**Checkpoint**: What table are events persisted to before processing?

---

## Step 10: Configuration System

**Goal**: Understand how configuration works.

### 10.1 Config Package
EverShop uses the `config` npm package. Configuration is accessed via:
```
packages/evershop/src/lib/util/getConfig.ts
```

### 10.2 Configuration Structure
Common config paths:
- `shop.*` - Shop settings
- `system.*` - System settings (database, session, extensions)
- `catalog.*` - Catalog module settings
- `checkout.*` - Checkout settings
- `pricing.*` - Pricing rules

### 10.3 Environment-Specific Config
Configuration can be environment-specific using `config/default.json`, `config/production.json`, etc.

### 10.4 ALLOW_CONFIG_MUTATIONS
Notice this environment variable is set in:
- `dev` script
- `test` script
- `start` environment

This allows runtime config modifications.

**Checkpoint**: What function should you use to access configuration values?

---

## Step 11: Testing Strategy

**Goal**: Understand the testing approach.

### 11.1 Jest Configuration
Tests are configured in:
```
jest.config.js (at root)
```

Key points:
- Tests must be compiled first (run from `dist/`)
- Pattern: `**/dist/**/tests/**/unit/**/*.test.[jt]s`
- Source tests in `src/` are excluded from test runs

### 11.2 Writing Tests
Tests live alongside source code:
```
packages/evershop/src/modules/<module>/tests/unit/
```

Example structure:
```
services/
  product.js
  tests/
    unit/
      product.test.js
```

### 11.3 Running Tests
```bash
# Must compile first!
npm run compile
npm run compile:db
npm test
```

**Checkpoint**: Why must tests be compiled before running?

---

## Step 12: Build System & Compilation

**Goal**: Understand the build process.

### 12.1 SWC Compilation
The project uses SWC for fast compilation:

```bash
# Compile evershop package
npm run compile

# Compile postgres-query-builder
npm run compile:db
```

Output goes to:
- `packages/evershop/dist/`
- `packages/postgres-query-builder/dist/`

### 12.2 Build for Production
```bash
npm run build
```

This runs `dist/bin/build` and prepares the application for production.

### 12.3 Development vs Production
- **Development**: Uses `src/` with hot reloading via `npm run dev`
- **Production**: Uses `dist/` with `NODE_ENV=production`

### 12.4 Package Publishing
The `@evershop/evershop` package uses `prepack` to run TypeScript compilation and copy assets before publishing to npm.

**Checkpoint**: What tool is used for compilation (hint: not tsc by default)?

---

## Step 13: Advanced Topics

Once you're comfortable with the basics, explore:

### 13.1 Extensions Development
Create a custom extension in:
```
extensions/my-extension/
```

Enable it in config:
```json
{
  "system": {
    "extensions": [
      {
        "name": "my-extension",
        "enabled": true,
        "path": "extensions/my-extension"
      }
    ]
  }
}
```

### 13.2 Widget System
Explore:
```
packages/evershop/src/lib/widget/
```

### 13.3 Cron Jobs
Look for cron job implementations in:
```
packages/evershop/src/lib/cronjob/
```

### 13.4 Payment Gateways
Study existing payment implementations:
```
packages/evershop/src/modules/stripe/
packages/evershop/src/modules/paypal/
packages/evershop/src/modules/cod/  (Cash on Delivery)
```

### 13.5 Theme System
Themes can be customized in:
```
themes/
```

---

## Learning Path Summary

1. **Week 1**: Steps 1-3 (Architecture, Modules, Bootstrap)
2. **Week 2**: Steps 4-6 (GraphQL, Database, API)
3. **Week 3**: Steps 7-9 (Frontend, Services, Events)
4. **Week 4**: Steps 10-12 (Config, Testing, Build)
5. **Ongoing**: Step 13 (Advanced topics, building extensions)

---

## Practical Exercises

### Exercise 1: Trace a Request
Follow a request through the system:
1. Start with an API call (e.g., GET /api/products)
2. Trace through: route.json → middleware → handler → service → database
3. Trace the response back

### Exercise 2: Add a Simple Feature
Add a new field to products:
1. Add GraphQL type
2. Add database migration
3. Update service layer
4. Update API endpoint
5. Test it

### Exercise 3: Create an Extension
Create an extension that:
1. Adds a new API endpoint
2. Emits a custom event
3. Has a subscriber that handles the event
4. Adds a new page to the admin panel

---

## Key Takeaways

1. **Modular Architecture**: Everything is a module (core or extension)
2. **Convention over Configuration**: File locations and naming matter
3. **GraphQL + REST**: GraphQL for complex queries, REST for simple operations
4. **Event-Driven**: Use events for async processing
5. **Type Safety**: TypeScript with compiled output in `dist/`
6. **Database-First**: PostgreSQL with query builder pattern

---

## Resources

- Official Docs: https://evershop.io/docs/development/getting-started/introduction
- This repo's AGENTS.md - Quick reference for contributors
- This repo's docs/ARCHITECTURE.md - Detailed architecture diagrams

Happy learning! 🚀
