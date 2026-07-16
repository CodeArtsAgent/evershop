# EverShop Core - Project Instructions for Claude

This is the **EverShop core repository** - a modular monolith eCommerce platform built on Express + React (SSR) + PostgreSQL + GraphQL. The package source lives at `packages/evershop/src/` and is published as `@evershop/evershop`.

The author and maintainer is The Nguyen (`support@evershop.io`).

## Stack
- **Runtime:** Node.js >= 20, Express, React 17 with SSR + hydration
- **Database:** PostgreSQL (no other DBs supported; SQL is plain Postgres)
- **GraphQL:** schema assembled at startup from per-module `.graphql` files
- **Bundler:** webpack 5 with SWC for transforms (no Babel)
- **Forms:** react-hook-form (wrapped by `components/common/form/Form.tsx`)
- **Styling:** Tailwind v4 + PostCSS + custom plugins
- **Query builder:** `@evershop/postgres-query-builder` (separate workspace package at `packages/postgres-query-builder/`)

## Application type
Multi-page application (MPA) - each route gets its own bundle and a full HTML response. Hydrated on the client. Not a SPA - there is no client-side router.

## Development commands

```bash
npm run compile       # SWC: src/ -> dist/ (required before tests and runtime)
npm run compile:db    # SWC: postgres-query-builder src/ -> dist/
npm run build         # webpack production build (reads from dist/)
npm run dev           # dev server with HMR (reads from dist/)
npm run start         # production start (reads from dist/)
npm run lint          # eslint --fix on packages/
npm test              # Jest unit tests
npm run test:e2e      # Playwright E2E tests (in tests/e2e/)
```

### Testing gotchas
- **Tests run from `dist/`, not `src/`.** `jest.config.js` sets `testMatch: ["**/dist/**/tests/**/unit/**/*.test.[jt]s"]` and ignores `src/` entirely. You must run `npm run compile` (and `npm run compile:db` for query-builder tests) before `npm test` or zero tests will execute.
- Jest maps `@evershop/postgres-query-builder` to `packages/postgres-query-builder/dist/index.js` - that dist must exist.
- Run a single test file: `npm test -- <path-relative-to-dist>` (e.g. `npm test -- modules/oms/tests/unit/status.test.js`).
- E2E tests live in `tests/e2e/` (separate package with its own `package-lock.json`). Run via `npm run test:e2e` from the repo root.

### CI and hooks
- CI (`.github/workflows/build_test.yml`): `npm install` -> `npm run compile` -> `npm run compile:db` -> `npm test`. **CI does not run lint.** Tested on Node 20 and 22.
- `husky` pre-commit (`.husky/pre-commit`): lint is **commented out** (`#FORCE_COLOR=1 npm run lint`). Do not assume lint runs automatically on commit.
- `eslint.config.js` ignores tests, extensions, dist, themes, public, media. `no-console` is `error`. `import/order` is `warn` with `newlines-between: never`.

### Configuration
- Uses the `config` npm package. See `configExample.text` for the full shape (shop settings, OMS status definitions, PSO mapping, carriers, theme config).
- OMS order/payment/shipment statuses and the payment-shipment -> order (`psoMapping`) are **config-driven**, not hardcoded. See `modules/oms/services/statusManager.ts`.
- Environment: `.env` file (DB connection, etc.). `tests/e2e/.env.example` shows E2E requirements.

## File / folder conventions
- **Migrations:** `Version-X.Y.Z.ts` (hyphen, not underscore) in `<module>/migration/`
- **Routes:** folder name = route ID, alphabetic only (a-z, A-Z), `route.json` declares the route
- **Middleware:** lowercase first letter, bracket-syntax ordering `[after]name[before].ts`
- **Master components:** uppercase first letter, `.tsx`, optional `export const layout = { areaId, sortOrder }`
- **Shared between routes:** folder named `routeA+routeB/` (e.g. `productEdit+productNew/`)
- **Site-wide middleware:** `pages/admin/all/`, `pages/frontStore/all/`, `pages/global/`, `api/global/`
- **Subscribers:** `subscribers/<event_name>/<handler>.ts`
- **Modules:** core in `packages/evershop/src/modules/`, user extensions in `extensions/` (project root, created on demand)
- **Module ID:** must be unique across the system

## Bootstrap is a hard wall
The hook system and registry are **locked** after every module's `bootstrap.ts` runs. Calling `addProcessor`, `hookBefore`, `hookAfter`, `registerWidget`, `registerJob`, `registerPaymentMethod`, etc. from inside a middleware or request handler **throws**. Always register from `bootstrap.ts`.

## Public import paths
Defined in `packages/evershop/package.json` `exports`. The most common ones:

```ts
import { select, insert, update, del, insertOnUpdate, startTransaction } from '@evershop/evershop/lib/postgres/query';
import { pool, getConnection } from '@evershop/evershop/lib/postgres';
import { addProcessor, getValue } from '@evershop/evershop/lib/util/registry';
import { hookBefore, hookAfter, hookable } from '@evershop/evershop/lib/util/hookable';
import { emit } from '@evershop/evershop/lib/event';
import { createSubscriber } from '@evershop/evershop/lib/event/subscriber';
import { buildUrl, buildAbsoluteUrl } from '@evershop/evershop/lib/router';
import { registerWidget } from '@evershop/evershop/lib/widget';
import { setContextValue, getContextValue } from '@evershop/evershop/graphql/services';
```

Module-level service entrypoints: `@evershop/evershop/oms/services`, `@evershop/evershop/checkout/services`, `@evershop/evershop/catalog/services`, etc.

## Common pitfalls
- `import { Request, Response } from 'express'` - wrong. Use `EvershopRequest`/`EvershopResponse` from `@evershop/evershop/types/request` and `types/response`.
- `module.exports` - wrong. ESM. Use `export default`.
- MySQL syntax - wrong. PostgreSQL. Use `IDENTITY` or `SERIAL`, double-quoted identifiers, `JSONB`, `gen_random_uuid()`.
- `migrations/` (plural) folder - wrong. Singular `migration/`.
- `Version_1.0.0.ts` (underscore) - wrong. Hyphen: `Version-1.0.0.ts`.
- `pages/frontend/` - wrong. `pages/frontStore/`.
- Arrow functions in hookable / processor callbacks when context is needed - `this` is bound via `.call()`, arrow functions can't access it.
- Adding a hook or processor from a middleware - locked after bootstrap, throws.
- New `.js` files in new code paths. The codebase has mixed `.js` and `.ts` because the migration to TypeScript is incremental, but **new authorship is `.ts`** (or `.tsx` for React). Editing an existing `.js` keeps it `.js` for small changes; full rewrites are a good moment to switch.

## Common mistakes (runtime / integration traps)

These compile cleanly and fail later. Each has bitten the codebase.

- **API handler that sends a response with a 2-arg `(request, response)` signature -> `ERR_HTTP_HEADERS_SENT`.** The framework inspects `function.length`; a 2-arg handler is treated as passive, so it auto-calls `next()` after your function resolves and `apiResponse` tries to send headers again. If you call `response.json()` / `response.send()` / `response.redirect()`, declare the third `next` parameter even if you never call it - the 3-arg signature disables auto-next.
- **Chaining `.where()` or `.orderBy()` directly off `.on()` in a query-builder JOIN -> `where is not a function` at runtime.** `.on()` returns a `Node`; `.where()` and `.orderBy()` live on `Query`/`SelectQuery`, not `Node`. Hold the query handle in a variable and call `.where()` / `.orderBy()` on it separately.
- **Passing `(column, alias)` to the top-level `select(...)` -> silent column rename -> `column "X" does not exist` at runtime.** The top-level `select(...)` is *variadic over columns* - `select('foo.uuid', 'method_uuid')` treats both strings as columns, not as `(column, alias)`. Only the chained `.select(col, alias)` form supports aliasing. Use `select().from(table).select(col, alias)` instead.
- **Passing `{ isSQL: true, value: '...' }` to `.given()` for raw SQL in UPDATE/INSERT -> `invalid input syntax for type ...` at runtime.** `UpdateQuery.given` / `InsertQuery.given` call `toString(value)` on every entry, which JSON-stringifies object values. The `{isSQL, value}` raw-escape convention is **only** honored inside `.where()` / `Leaf` / `RawLeaf`, not for SET / VALUES. When you need raw SQL on the write side (`COALESCE(col, NOW())`, `col + 1`, `gen_random_uuid()`), drop to `connection.query()` with bind parameters for just the user values. Canonical pattern: `modules/oms/services/updateShipmentStatus.ts:78-98`.
- **`.execute(connection)` / `.load(connection)` on a fresh `getConnection()` PoolClient before `startTransaction(connection)` -> `Release called on client which has already been released to the pool` at runtime.** The query-builder's internal `release()` only short-circuits when `connection.INTRANSACTION === true`, a flag exclusively set by `startTransaction`. Pre-tx reads on a freshly-acquired PoolClient auto-release it back to the pool, and the next `startTransaction(connection)` operates on a detached client. Rule: either call `startTransaction` IMMEDIATELY after `getConnection`, or run pre-tx reads on the shared `pool` (which `release()` ignores) and acquire the dedicated PoolClient only at the top of the tx. The canonical "delayed tx" pattern (necessary when an external network call has to run between read and write - e.g. `carrier.createLabel()`) is in `modules/oms/services/createShipment.ts:477-557`.
- **Hook called after a conditional early return in a React component -> `Rendered more hooks than during the previous render`.** All hooks must run in the same order on every render. If one render takes an early return before a `useEffect` and the next render reaches it, React errors. Move every hook above any conditional return.
- **`.admin.graphql` type referenced from a non-admin `.graphql` file -> "Unknown type X" at storefront schema build.** `buildStoreFrontSchema` filters out `.admin.graphql`; types defined there aren't visible to the storefront schema. Either move the type to a non-admin file or mark the referencing file `.admin.graphql` too. The two schemas build separately - admin sees both, storefront sees only non-admin.
- **Dropping a DB column without grepping across modules.** EverShop modules share tables. A resolver in `modules/base/` may read a column owned by `modules/checkout/`. When dropping a column, `grep -rn "table\\.column" packages/evershop/src` across the whole tree, not just the owning module.
- **`hookable()` keys hooks by the wrapped function's `.name`, so a `…Impl` declaration silently kills its public hooks.** `hookable(fooImpl)` registers under `'fooImpl'`, but a `hookBeforeFoo` helper that calls `hookBefore('foo', …)` registers under `'foo'` - they never meet, the hook never fires, and nothing errors (the wrapped function still runs, the transaction still commits). Wrap a **named function expression** whose intrinsic name *is* the hook key, even if the binding differs: `const fooImpl = async function foo() {…}` (the `checkout.ts:10` idiom - `const _checkout = async function checkout(`). A plain `function fooImpl() {}` declaration sets `.name = 'fooImpl'` and breaks it. Guard test: `modules/oms/tests/unit/hookNameAlignment.test.js`.
- **A widget `settingComponent` that reads a list setting as `watch('settings.x') ?? initial` works in the page-builder drawer but throws `items.map is not a function` on the legacy `/admin/widgets/edit` page.** The two surfaces seed settings differently: the drawer's page-level form holds real arrays/objects, but the legacy `<Form>` seeds list fields as a JSON **string** via a hidden `defaultValue={JSON.stringify(...)}` input - and `??` only guards null, so the string reaches `RepeatableAccordion`. Read list settings with `useArraySetting('settings.x', initial)` and mutate via `asArray(getValues('settings.x'), initial)` (both from `@components/common/page-builder`), or hold the array with `useFieldArray` like `SlideshowSetting`.

## Code search strategy

Choose the right search tool for the task rather than always defaulting to one:

- **CodeSemanticSearch** - for "how/where/what" questions, exploring unfamiliar code, finding code by meaning when you don't know the exact symbol name. Reuse the user's exact wording when possible.
- **CodeGraphSearch** - for tracing call chains, dependencies, impact analysis ("what breaks if I change X?"), and execution paths. Use `graph_depth=1` for direct callers/callees, `2` for transitive context (default), `3` for broad impact analysis. Extract symbols from the query first - if none match, it degrades to semantic matching.
- **grep** - for exact symbol lookups, exact string/pattern matching, counting occurrences, and searching for specific imports/exports when you know the name. Also the fallback when semantic/graph search returns no results.
- **glob** - for finding files by name pattern (e.g., `**/*.graphql`, `src/**/*.ts`) or locating files by naming convention. Use over `ls` for file discovery.
- **read** - for reading known file paths or specific line ranges. Prefer over re-reading chunks already returned by search tools.
- **Combine tools** when needed: `glob` to locate files -> `grep` for exact symbols inside them -> `CodeSemanticSearch` for behavior questions -> `CodeGraphSearch` for impact/dependency tracing.
- **Break down large questions** into smaller, focused searches. Don't combine multiple intents (e.g., "what is X and how does it dispatch events?") in one query - split them and run in parallel.
- **Avoid** single-word queries (e.g., just `AuthService`) for semantic/graph search - use a full question instead (e.g., "How does AuthService authenticate users?").

## Doing work in this repo
- The published package is built from `src/` to `dist/` via SWC (`npm run compile`). Runtime loads `.js` from `dist/`. When editing, edit `.ts` in `src/`.
- Never bypass `husky` hooks (`--no-verify`) or skip type/lint failures without an explicit go-ahead.
- `packages/postgres-query-builder/` is a separate workspace package. Its source is a single file: `src/index.ts`. It exports `select`, `insert`, `update`, `del`, `startTransaction`, etc. The evershop package re-exports these from `lib/postgres/query.ts`.
