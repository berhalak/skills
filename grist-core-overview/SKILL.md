---
name: grist-core-overview
description: Use when starting work on the Grist codebase, onboarding to the project, needing to understand how major pieces connect, figuring out where to make changes for a new feature or bugfix, or when unsure which files/directories own a given concern. Also use when someone asks "how does Grist work" or "where does X live".
---

# Grist Core — Project Overview & Architecture

## What Grist Is

Open-source spreadsheet-database hybrid. Combines spreadsheet UI with relational database, Python formulas, access control, and REST API. Documents are SQLite files. Formulas run in a sandboxed Python process. Full-stack TypeScript (Node.js server + grainjs client) + Python.

## Tech Stack Summary

| Layer | Tech |
|-------|------|
| Server | Node.js, Express, WebSocket (`Comm`) |
| Client | grainjs (reactive DOM), Knockout (legacy), Backbone (legacy models) |
| Bundler | Webpack |
| Document DB | SQLite (one file per document) |
| Home DB | SQLite or PostgreSQL via TypeORM |
| Formula engine | Python 3 in gVisor/Pyodide sandbox |
| Tests | Mocha + Selenium WebDriver (nbrowser), chai |
| Package manager | Yarn v1 |

## Repository Map

```
app/
  client/          Browser code (grainjs UI, components, models, widgets)
  server/          Node.js server (Express, WebSocket, doc management)
    MergedServer.ts   Main entry — composes home+docs+static servers
    lib/              Server core (~175 files)
      FlexServer.ts   Central Express app, all routes/middleware (~2000 lines)
      ActiveDoc.ts    In-memory document: formula eval, action application
      DocApi.ts       REST API for document operations (rows/columns/tables)
      DocManager.ts   Maps doc IDs → ActiveDoc instances
      DocStorage.ts   SQLite document persistence
      NSandbox.ts     Spawns/communicates with Python sandbox
      Authorizer.ts   Authentication, user extraction, access checks
      Comm.ts         WebSocket server for real-time collaboration
      ICreate.ts      Build-flavor DI interface
  common/          Shared code (~142 files: types, DocActions, URLs, ACL)
    schema.ts        AUTO-GENERATED from Python — never edit manually
    DocActions.ts    Action types (fundamental protocol)
  gen-server/      Home database layer
    entity/          TypeORM entities (User, Org, Document, etc.)
    migration/       ~50 TypeORM migrations
    lib/homedb/      HomeDBManager (~5000+ lines, all home DB queries)
    ApiServer.ts     REST API for orgs/workspaces/docs CRUD
  plugin/          Plugin API types (for custom widgets)
sandbox/
  grist/           Python formula engine (engine.py, main.py, schema.py)
stubs/
  app/server/lib/create.ts   CoreCreate (OSS build entrypoint)
test/
  nbrowser/        Selenium integration tests (~188 files)
  server/          Server tests
  gen-server/      Home DB tests
  client/          Client unit tests
buildtools/        Build scripts, webpack config, code generators
static/            Built webpack bundles + assets
```

## Architecture — How Pieces Connect

### Build Flavors & DI

Three builds export different `create: ICreate` singletons from `create.ts`:
- **Core** (`stubs/`): `CoreCreate` — OSS, no billing, no external storage
- **Enterprise** (`ext/`): `EnterpriseCreate` — activation-key billing, external storage
- **SaaS** (`app/` overlay): `SaaSCreate` — Stripe billing, per-team products

`create` provides factories for: Billing, Notifier, AuditLogger, Telemetry, Assistant, Sandbox, LoginSystem, StorageManager.

### Server Startup

`MergedServer.create()` → instantiates `FlexServer` → incrementally adds capabilities based on which server types are enabled (`home`, `docs`, `static`, `app`). Methods MUST be called in specific order.

### Document Lifecycle

`DocManager` → `ActiveDoc` (in-memory) → `DocStorage` (SQLite) ↔ Python `engine.py` (sandbox, via stdin/stdout marshal)

### The Action Protocol (Critical)

ALL document mutations use: `UserAction[]` → Python sandbox → `DocAction[]`. Never bypass this — it powers undo/redo, ACL, replication, and real-time sync.

### Client-Server Communication

- **REST**: Express routes in `DocApi.ts` + `ApiServer.ts`
- **WebSocket**: `Comm.ts` (both sides) for real-time doc collaboration
- **Page serving**: `AppEndpoint.ts` → `sendAppPage()` renders Handlebars templates → injects `GristLoadConfig` → browser loads `main.bundle.js` → `app.js` → `AppImpl`

## Where To Make Changes

### Add REST API endpoint (document-level)
1. Route in `app/server/lib/DocApi.ts`
2. Types in `app/common/ActiveDocAPI.ts` or `app/plugin/DocApiTypes.ts`
3. Client API in `app/common/UserAPI.ts` → `DocAPIImpl`
4. Test in `test/server/DocApi.ts` or `test/nbrowser/`

### Add REST API endpoint (home-level)
1. Route in `app/gen-server/ApiServer.ts`
2. Query/mutation in `HomeDBManager.ts`
3. Client API in `app/common/UserAPI.ts` → `UserAPIImpl`
4. Test in `test/gen-server/`

### Add UI page
1. Component in `app/client/ui/MyPage.ts` (grainjs)
2. CSS in `app/client/ui/MyPageCss.ts` (styled pattern)
3. Route in `app/client/ui/AppUI.ts`
4. Server endpoint in `FlexServer.ts` if needed
5. `sendAppPage()` call in `AppEndpoint.ts`

### Add document schema field
1. Edit `sandbox/grist/schema.py` (source of truth)
2. Run `npm run generate:schema:ts`
3. Migration in `sandbox/grist/migrations.py`
4. Update `DocModel.ts` entity record on client

### Add home DB column/table
1. Modify entity in `app/gen-server/entity/`
2. New migration in `app/gen-server/migration/<timestamp>-<Name>.ts`
3. Update `HomeDBManager.ts`

## Key Commands

```bash
npm run start              # Dev server with watch
npm run build              # tsc + webpack (dev mode)
yarn tsc --build           # Type check only
GREP_TESTS="X" npm test   # Run specific tests
npm run test:nbrowser      # All Selenium tests
npm run lint               # ESLint
npm run generate:schema:ts # Regen schema.ts from Python
```

## Sharp Edges

- **`FlexServer.ts`**: Monolithic, order-dependent startup. Changes affect everything.
- **`HomeDBManager.ts`**: ~5000+ lines, complex permission queries. Easy to introduce ACL bugs.
- **`schema.ts`** and `*-ti.ts`**: Auto-generated. Never edit manually.
- **Mixed observables**: grainjs + Knockout coexist, interop via `fromKo()`/`toKo()`. Subtle disposal bugs possible.
- **nbrowser tests**: Selenium + async UI = flakiness. Keep waits ≤1s on localhost.
- **Action system**: Bypassing `UserAction → Python → DocAction` pipeline breaks undo, ACL, replication.
- **Three build flavors**: Code must work in core, enterprise, and SaaS. Some paths only tested in specific builds.

## Conventions

- Files: PascalCase for classes, camelCase for utilities
- CSS-in-JS: `*Css.ts` files using grainjs `styled()`
- Interfaces: `I`-prefix for DI (`ICreate`, `ISandbox`)
- Localization: `makeT("ModuleName")`
- Import boundaries: `common/` → no client/server imports; `client/` → no server imports
- `-ti.ts` files: auto-generated runtime type checkers
- All doc changes through action system, never direct SQLite writes

## Reading Order for Onboarding

1. `app/server/MergedServer.ts` — server startup
2. `app/server/lib/ICreate.ts` — DI pattern
3. `app/common/DocActions.ts` — action protocol
4. `app/server/lib/ActiveDoc.ts` (top 100 lines) — document lifecycle
5. `app/server/lib/DocApi.ts` (top 100 lines) — REST API patterns
6. `app/client/app.js` + `app/client/ui/App.ts` — client bootstrap
7. `app/client/models/DocModel.ts` — client data model

## Deep Reference

For detailed execution flows, auth flow specifics, full module inventories, testing patterns, debug workflows, and areas not fully investigated, see @reference.md in this skill's directory.
