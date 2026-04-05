# Module: `graphql`

## Purpose

The **graphql** module is **infrastructure**, not the business domain: it **aggregates** `.graphql` documents and **resolver modules** from all core modules and enabled **extensions**, builds an **executable schema**, and (via `services/buildSchema.js`) exposes helpers such as **`rebuildSchema`** for dynamic reload scenarios.

## How it works

1. **Type definitions** — `buildTypeDefs(isAdmin)` globs `MODULESPATH/*/graphql/types/**/*.graphql` plus each extension path, merges with `@graphql-tools/merge`, and **excludes `*.admin.graphql`** when `isAdmin === false` so the storefront schema stays smaller and safer.

2. **Resolvers** — `buildResolvers(isAdmin)` globs `*.resolvers.{js,ts}` with different ignore rules:

   - **Admin** — ignores `.ts` (expects compiled `.js` in production merge step; dev can import `.ts` via dynamic import path).
   - **Storefront** — ignores `.admin.resolvers.js` / `.admin.resolvers.ts`.

3. **Bootstrap** — `bootstrap.js` side-effect imports `buildSchema.js`, which eagerly builds the **admin** schema (`buildResolvers(true)`). Separate server or entrypoints may build storefront schemas with `isAdmin: false` (see HTTP handlers that serve GraphQL).

4. **Extensions** — `getEnabledExtensions()` adds paths to both globs so extensions behave like core modules.

## Framework implementation

| Mechanism | Location |
|-----------|----------|
| Type merge | `modules/graphql/services/buildTypes.js` — `buildTypeDefs` |
| Resolver merge | `modules/graphql/services/buildResolvers.js` — `buildResolvers` |
| Executable schema | `modules/graphql/services/buildSchema.js` — `makeExecutableSchema`, `rebuildSchema` |
| Bootstrap | `modules/graphql/bootstrap.js` |

## HTTP APIs

| Route | Method | Path | Role |
|-------|--------|------|------|
| `graphql` | GET, POST | `/graphql` | Storefront GraphQL endpoint |
| `adminGraphql` | GET, POST | `/admin/graphql` | Admin GraphQL endpoint (includes `.admin` types) |

## Database tables

**None.** The graphql module has no migrations — it is pure infrastructure.

## GraphQL types

| Type | File | Scope | Query fields |
|------|------|-------|-------------|
| `Query` (root) | `Query.graphql` | Shared | `hello` (root type that all modules extend) |

All other modules contribute types by placing `.graphql` files and `.resolvers.js` under their own `graphql/types/` directories; this module merges them.

## User flows

### Schema build process (startup)

```mermaid
flowchart TD
    A[graphql/bootstrap.js] --> B[import buildSchema.js]
    B --> C[buildTypeDefs isAdmin=true]
    B --> D[buildResolvers isAdmin=true]

    C --> E[Glob: MODULESPATH/*/graphql/types/**/*.graphql]
    C --> F[Glob: extensions/*/graphql/types/**/*.graphql]
    E & F --> G[mergeTypeDefs via @graphql-tools/merge]

    D --> H[Glob: **/*.resolvers.js,ts]
    H --> I{isAdmin?}
    I -->|Admin| J[Include all resolvers, ignore .ts]
    I -->|Storefront| K[Exclude .admin.resolvers.*, ignore .ts]
    J & K --> L[mergeResolvers]

    G & L --> M[makeExecutableSchema]
    M --> N[Exported schema used by HTTP endpoints]
```

### GraphQL request flow

```mermaid
sequenceDiagram
    participant Client
    participant Endpoint as POST /graphql or /admin/graphql
    participant Middleware as Auth + context middleware
    participant Executor as GraphQL executor
    participant Resolvers as Module resolvers
    participant DB as PostgreSQL

    Client->>Endpoint: POST {query, variables}
    Endpoint->>Middleware: Validate auth (JWT/session)
    alt Admin endpoint
        Middleware->>Middleware: Use admin schema (includes .admin types)
    else Storefront endpoint
        Middleware->>Middleware: Use storefront schema (no .admin types)
    end
    Middleware->>Executor: Execute query against schema
    Executor->>Resolvers: Resolve fields
    Resolvers->>DB: SQL queries via query builder
    DB-->>Resolvers: Data
    Resolvers-->>Executor: Resolved data
    Executor-->>Client: JSON {data, errors?}
```

### Schema split: admin vs storefront

```mermaid
flowchart LR
    subgraph All modules
        A[*.graphql files]
        B[*.admin.graphql files]
        C[*.resolvers.js files]
        D[*.admin.resolvers.js files]
    end

    subgraph Admin schema
        A --> E[Included]
        B --> E
        C --> F[Included]
        D --> F
    end

    subgraph Storefront schema
        A --> G[Included]
        B -.->|Excluded| G
        C --> H[Included]
        D -.->|Excluded| H
    end
```

## What could be done better

- **Admin vs storefront clarity** — `.admin.graphql` / `.admin.resolvers` convention is easy to violate; a CI check that fails if admin-only fields leak into storefront typeDefs would help.

- **TypeScript resolver story** — Production ignores `.ts` in some branches; document the required **compile** step for resolver `.ts` files so extensions do not break deploys.

- **Schema size and cold start** — Large merges cost startup time; optional lazy loading or subgraphs (federation) could help very large installs.

- **Deprecation policy** — GraphQL `@deprecated` on fields with migration notes helps theme and extension authors.

- **Persisted queries / complexity limits** — For public storefront endpoints, query depth/complexity limits mitigate abuse.
