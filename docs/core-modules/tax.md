# Module: `tax`

## Purpose

**tax** adds **tax class** administration, **per line-item tax percentage** resolution on the cart, and **pricing.tax** configuration (rounding mode, precision, round at line vs unit vs total, and whether catalog prices include tax).

## How it works

1. **Cart item field** — `registerCartItemTaxPercentField` attaches at priority `0` on `cartItemFields` so tax can be computed early in the pipeline relative to other item fields (promotion uses `11`; exact interactions depend on field ordering after `sortFields`).

2. **Configuration** — Merges `pricing.tax` into `configurationSchema` and sets `config.util.setModuleDefaults('pricing', { tax: ... })` with defaults such as `price_including_tax: true`.

3. **Admin lists** — `taxClassCollectionFilters` + pagination filters for GraphQL tax class queries.

4. **GraphQL + migrations** — Tax classes and rules stored in Postgres; types exposed for admin.

## Framework implementation

| Mechanism | Location |
|-----------|----------|
| Bootstrap | `modules/tax/bootstrap.js` |
| Item field | `services/registerCartItemTaxPercentField.js` |
| Filters | `registerDefaultTaxClassCollectionFilters.js` |

**catalog** sets general `pricing.rounding` / `precision`; **tax** nests tax-specific behavior under `pricing.tax`.

## HTTP APIs

| Route | Method | Path | Role |
|-------|--------|------|------|
| `createTaxClass` | POST | `/tax/classes` | Create tax class |
| `updateTaxClass` | PATCH | `/tax/classes/:id` | Edit tax class |
| `createTaxRate` | POST | `/tax/classes/:class_id/rates` | Add rate to class |
| `updateTaxRate` | PATCH | `/tax/rates/:id` | Edit rate |
| `deleteTaxRate` | DELETE | `/tax/rates/:id` | Remove rate |
| `taxSetting` | GET | `/setting/tax` | Admin settings page |

## Database tables

| Table | Migration | Role |
|-------|-----------|------|
| `tax_class` | `Version-1.0.0` | Tax class definitions |
| `tax_rate` | `Version-1.0.0` | Rate rules (country, province, percentage) per class |

Also alters `product` table to add FK `FK_TAX_CLASS` → `tax_class`.

## GraphQL types

| Type | File | Scope | Query fields |
|------|------|-------|-------------|
| `TaxSetting` | `TaxSetting.graphql` / `.admin` | Shared + Admin | *(extends Setting)* |
| `TaxClass`, `TaxRate`, `TaxClassCollection` | `TaxClass.admin.graphql` | Admin | `taxClasses`, `taxClass` |

## User flows

### Tax configuration (admin)

```mermaid
sequenceDiagram
    participant Admin
    participant TaxPage as GET /setting/tax
    participant ClassAPI as Tax Class APIs
    participant RateAPI as Tax Rate APIs
    participant DB as tax_class + tax_rate

    Admin->>TaxPage: Navigate to tax settings
    Admin->>ClassAPI: POST /tax/classes {name: "Standard"}
    ClassAPI->>DB: INSERT INTO tax_class
    ClassAPI-->>Admin: Tax class created
    Admin->>RateAPI: POST /tax/classes/:classId/rates {country, province, rate: 10, name: "VAT"}
    RateAPI->>DB: INSERT INTO tax_rate
    RateAPI-->>Admin: Rate added to class
    Note over Admin: Assign tax class to products via catalog module
```

### Tax calculation in cart pipeline

```mermaid
flowchart TD
    A[Cart item added] --> B[cartItemFields pipeline runs]
    B --> C[registerCartItemTaxPercentField priority 0]
    C --> D{Product has tax_class_id?}
    D -->|Yes| E[Look up tax_rate by class + shipping address country/province]
    D -->|No| F[Tax percent = 0]
    E --> G[Set tax_percent on cart item]
    F --> G
    G --> H[Later fields compute:]
    H --> I[line_total_incl_tax = price * qty * 1 + tax_percent/100]
    H --> J[tax_amount per line]
    J --> K[Cart total_tax_amount = sum of line taxes]

    style C fill:#f9f,stroke:#333
    style E fill:#bbf,stroke:#333
```

### Tax rounding behavior

```mermaid
flowchart LR
    A[Config: pricing.tax] --> B{round_level}
    B -->|unit| C[Round tax per unit, then multiply by qty]
    B -->|line| D[Compute line tax, then round]
    B -->|total| E[Sum raw line taxes, round once at cart total]
    C & D & E --> F{rounding mode}
    F -->|round| G[Math.round]
    F -->|floor| H[Math.floor]
    F -->|ceil| I[Math.ceil]
```

## What could be done better

- **Field priority semantics** — Document the intended order: base price → tax → promotions or the reverse, depending on business rules; mismatches here cause real money bugs.

- **Jurisdictional complexity** — US sales tax, EU VAT, and GST often need address + product class matrix; the module may need plugin hooks beyond a single percentage field.

- **Display vs calculation** — Storefront “including tax” labeling should align with `price_including_tax`; centralize formatting helpers to avoid drift across themes.

- **Integration tests** — Property-style tests for rounding (`round_level`: unit/line/total) catch floating-point issues.
