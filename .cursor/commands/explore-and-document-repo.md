# Explore & document repository

You are documenting this repository for a developer who is new to the project.

## Scope

- Explore the codebase systematically (entry points, packages/workspaces, configuration, build, runtime).
- Identify all major **user-visible features** (CLI, APIs, UIs, integrations) and **internal capabilities** (events, jobs, extensions).

## Deliverables (use clear headings)

1. **Executive summary** — what the product is, main tech stack, how it runs locally.
2. **Repository map** — top-level folders, monorepo/workspaces if any, where “real” app code lives vs generated/vendor.
3. **Runtime & entry points** — main binaries, server bootstrap, config loading, env vars that matter.
4. **Core modules / domains** — list each major area with: purpose, key files, how it plugs into the rest.
5. **Data & persistence** — DB/migrations, key entities at a high level (no secrets).
6. **Public surfaces** — HTTP routes, GraphQL, webhooks, CLIs; point to representative files.
7. **Cross-cutting concerns** — auth, logging, errors, i18n, caching, events, background work.
8. **How to change things safely** — compile/build/test commands used in this repo; what to run before a PR.

## Diagrams (required)

Include **Mermaid** diagrams (readable in GitHub/GitLab and many editors):

- **System context** — this repo vs external systems (DB, queues, payment, etc.).
- **High-level architecture** — main processes/packages and how they connect.
- **Request or job flow** — one end-to-end path (e.g. HTTP request → handler → DB → response, or CLI → services).
- **Module / package dependency** — boxes for major packages and arrows showing dependencies or data flow.

Use simple shapes and short labels; prefer accuracy over detail.

## Exploration rules

- Prefer reading real files and tracing imports/calls over guessing.
- Cite important paths and symbols with fenced code blocks using the format `startLine:endLine:filepath` (opening fence on its own line).
- If something is uncertain, say what you checked and what remains unknown.
- Do not invent features; ground every claim in the code or checked-in docs (`README`, `AGENTS.md`, etc.).
- Keep the doc maintainable: note “source of truth” files for each topic.

Start by listing what you will open first (entry configs, `package.json` workspace layout, main app bootstrap), then execute the exploration and write the final structured document with diagrams.
