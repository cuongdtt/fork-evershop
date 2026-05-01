# Project Overview and PDR (Product Development Requirements)

## Table of Contents

1. [What is EverShop](#what-is-evershop)
2. [Core Capabilities and Features](#core-capabilities-and-features)
3. [Target Users and Use Cases](#target-users-and-use-cases)
4. [Product Vision and Roadmap Direction](#product-vision-and-roadmap-direction)
5. [Development Requirements and Constraints](#development-requirements-and-constraints)

---

## What is EverShop

EverShop is an **open-source e-commerce platform** built with modern web technologies. It provides a complete shopping cart solution for online merchants, featuring a storefront for customers, an admin dashboard for store management, and a flexible extension system for customization.

### Product Description

EverShop is a full-stack e-commerce platform that enables merchants to build and manage online stores. It combines a Node.js/Express backend with React-based frontend interfaces, using PostgreSQL as the primary database and GraphQL for API communication.

### Licensing

EverShop is released under the **GNU General Public License v3.0 (GPL-3.0)**. This copyleft license ensures that any derivative works must also be distributed under the same license terms, maintaining the open-source nature of the project.

- **packages/evershop**: GPL-3.0
- **packages/postgres-query-builder**: MIT (separate query builder library)

### Tech Stack

| Layer | Technology |
|-------|-------------|
| Runtime | Node.js (18+) |
| Web Framework | Express.js |
| Database | PostgreSQL |
| API Layer | GraphQL (primary), REST (legacy) |
| Frontend | React 17.x |
| Build Tools | SWC, Webpack, TypeScript |
| Package Manager | pnpm (workspaces) |

---

## Core Capabilities and Features

### 1. Product Catalog Management

- Product creation with variants (size, color, etc.)
- Category organization
- Attribute management
- Inventory tracking
- Pricing with tax calculation support

### 2. Shopping Cart and Checkout

- Persistent cart functionality
- Multi-step checkout process
- Guest checkout support
- Multiple payment gateways (Stripe, PayPal, Cash on Delivery)
- Order processing and confirmation

### 3. Customer Management

- Customer registration and login
- Address book management
- Order history viewing
- Customer groups and segmentation

### 4. Order Management System (OMS)

- Order creation and tracking
- Order status management
- Shipment tracking
- Refund and return processing
- Order history and reporting

### 5. Content Management System (CMS)

- Custom pages creation
- Widget system for page customization
- SEO metadata management
- URL rewriting for SEO-friendly URLs

### 6. Promotions and Discounts

- Coupon codes
- Percentage and fixed discounts
- Promotional campaigns

### 7. Admin Dashboard

- Product management interface
- Order management interface
- Customer management
- Store settings configuration
- Analytics and reporting

### 8. Storefront

- Responsive design
- Product browsing and search
- Category navigation
- Product detail pages
- Shopping cart UI

### 9. Extension System

- Custom extension development
- Module-based architecture
- Priority-based loading
- Theme customization support

---

## Target Users and Use Cases

### Primary Target Users

1. **Small to Medium E-commerce Merchants**
   - Businesses looking for an open-source alternative to Shopify, WooCommerce, or Magento
   - Merchants with technical capability to self-host and customize

2. **Developers and Agencies**
   - Development teams building custom e-commerce solutions
   - Agencies managing multiple client stores

3. **Technical Entrepreneurs**
   - Startups requiring a flexible, customizable e-commerce foundation
   - Businesses with specific customization requirements

### Common Use Cases

1. **Online Retail Stores**
   - Clothing and fashion e-commerce
   - Electronics and gadgets
   - Home goods and furniture
   - Food and beverages

2. **B2B E-commerce**
   - Wholesale ordering systems
   - Business-to-business marketplaces

3. **Marketplace Platforms**
   - Multi-vendor marketplace foundations
   - Niche community marketplaces

4. **D2C (Direct-to-Consumer) Brands**
   - Brand-owned online stores
   - Subscription-based products

---

## Product Vision and Roadmap Direction

### Vision Statement

EverShop aims to be the most flexible and developer-friendly open-source e-commerce platform, enabling merchants to create unique, high-performance online stores while maintaining full control over their data and customization.

### Current Focus Areas

1. **Core Platform Stability**
   - Improving test coverage and code quality
   - Performance optimization
   - Security enhancements

2. **Developer Experience**
   - Better documentation
   - Simplified extension development
   - Improved debugging tools

3. **Modern Frontend**
   - React 18 compatibility
   - Modern build tooling
   - Improved admin UI/UX

4. **E-commerce Features**
   - Enhanced checkout flows
   - Advanced promotion rules
   - Multi-currency and multi-language support

### Community-Driven Development

- Open contribution model via GitHub
- Focus on upstream contributions to the main `dev` branch
- Regular releases with changelog documentation

---

## Development Requirements and Constraints

### System Requirements

| Component | Minimum Version |
|-----------|-----------------|
| Node.js | 18.x or higher |
| PostgreSQL | 12.x or higher |
| pnpm | 7.x or higher |
| npm | 9.x (for CI) |

### Development Environment Setup

```bash
# Install dependencies
npm install

# Compile the platform (SWC)
npm run compile

# Compile the query builder
npm run compile:db

# Setup database and run migrations
npm run setup

# Start development server
npm run dev

# Run tests
npm run test

# Run linting
npm run lint
```

### Build Requirements

1. **Compilation Pipeline**
   - SWC for fast TypeScript/JavaScript compilation
   - TypeScript for type checking and npm publishing
   - Webpack for frontend bundle creation

2. **Testing Requirements**
   - Jest for unit testing
   - Tests must be in `dist/**/tests/**/unit/**/*.test.[jt]s`
   - Tests run against compiled code, not source

3. **Code Quality**
   - ESLint with TypeScript and React support
   - `no-console` rule set to error (use logger instead)
   - Prettier for code formatting

### CI/CD Requirements

- GitHub Actions workflow for pull requests
- Node.js versions 20 and 22 tested
- Required steps: install, compile, compile:db, test
- Lint is NOT in CI (run manually)

### Configuration Requirements

- Uses `node-config` package for configuration
- Default configuration in `config/default.json`
- Environment variables for sensitive data
- `ALLOW_CONFIG_MUTATIONS=true` required for dev and test

### Extension Development Constraints

- Extension names must not collide with core modules
- Extensions must have `enabled: true` in config
- Production requires `dist/` directory
- Development requires `src/` directory
- Sorted by `priority` (lower loads first)

### Git and Contribution Requirements

- Target branch: `dev`
- PRs require tests for new features/fixes
- Follow code style guidelines
- License: GPL-3.0 for contributions

---

## Summary

EverShop is a comprehensive open-source e-commerce platform that provides merchants with a flexible, customizable solution for online retail. With its modern tech stack (Node.js, Express, PostgreSQL, GraphQL, React), modular architecture, and extension system, it offers a solid foundation for building unique e-commerce experiences while maintaining full ownership and control.

The platform is actively maintained with regular updates, clear contribution guidelines, and a focus on both stability and feature development. Developers can extend functionality through the well-documented extension system, while merchants benefit from the comprehensive built-in features for catalog management, checkout, order processing, and store administration.