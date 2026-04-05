# Core commerce and operations modules

This folder documents EverShop’s **core modules** under `packages/evershop/src/modules/*`: what each area does, how the framework wires it (bootstrap, processors, hooks, GraphQL, HTTP APIs, pages), and pragmatic improvement ideas.

## Index

| Module | Doc |
|--------|-----|
| Foundations (shop config, i18n, global API behavior) | [base.md](./base.md) |
| Admin authentication | [auth.md](./auth.md) |
| Storefront customers | [customer.md](./customer.md) |
| Catalog and pricing defaults | [catalog.md](./catalog.md) |
| Cart and checkout | [checkout.md](./checkout.md) |
| Promotions and discounts | [promotion.md](./promotion.md) |
| Tax | [tax.md](./tax.md) |
| Stripe / PayPal / Cash on delivery | [payments.md](./payments.md) |
| Order management (OMS) | [oms.md](./oms.md) |
| CMS (pages, widgets, theme config) | [cms.md](./cms.md) |
| Store settings (database-backed) | [setting.md](./setting.md) |
| GraphQL infrastructure | [graphql.md](./graphql.md) |

## How core modules load

Core modules are enumerated in `packages/evershop/src/bin/lib/loadModules.js`. At startup, each module’s optional `bootstrap.js` (compiled from `bootstrap.js` or `bootstrap.ts` in source) runs via `loadBootstrapScript` in `packages/evershop/src/bin/lib/bootstrap/bootstrap.ts`. Routes, middleware, and GraphQL pieces are discovered from filesystem conventions under each module directory (see [graphql.md](./graphql.md) and project `AGENTS.md`).

Extensions use the same patterns when enabled in `system.extensions`.
