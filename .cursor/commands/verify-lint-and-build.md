# Verify lint & build-fast

Run this from the **repository root** after substantive code changes to confirm the tree still lints and builds.

## Steps

1. If `packages/evershop/dist` or `packages/postgres-query-builder/dist` is missing or you changed **source** under those packages, run first:
   - `pnpm compile`
   - `pnpm compile:db`
2. Run **`pnpm lint`** (root script fixes what ESLint can auto-fix).
3. Run **`pnpm build-fast`** (production build with `--skip-minify`).

## On failure

- Read the error output; fix the root cause in source (prefer the smallest change).
- If failures are from stale or missing compiled output, re-run `pnpm compile` and `pnpm compile:db`, then retry from step 2.
- Repeat until **`pnpm lint`** and **`pnpm build-fast`** both exit successfully.

## Done when

- `pnpm lint` completes with exit code 0.
- `pnpm build-fast` completes with exit code 0.

Do not mark the task finished until both commands succeed.
