# Project Skill

## 1. What This Project Is

Grist is an open-source, self-hostable spreadsheet-database hybrid. It combines a familiar spreadsheet UI with relational database capabilities, Python formulas, access control rules, and a REST API. Documents are stored as SQLite files, with a Python sandbox executing formulas. The architecture is a full-stack TypeScript/Node.js + Python system with three build flavors: Core (OSS), Enterprise (self-hosted with activation keys), and SaaS (hosted, Stripe-billed).

## 2. Tech Stack

| Layer | Technology |
|-------|-----------|
| **Language** | TypeScript (server + client), Python 3 (formula sandbox) |
| **Server** | Node.js, Express, WebSocket (custom `Comm` protocol) |
| **Client** | grainjs (reactive DOM library), Knockout.js (legacy), Backbone (legacy models) |
| **UI Framework** | Custom component system (no React/Vue/Angular) |
| **Bundler** | Webpack |
| **Database** | SQLite (per-document storage), SQLite or PostgreSQL (home/meta DB via TypeORM) |
| **ORM** | TypeORM (for the "home" database: orgs, users, workspaces, billing) |
| **Test** | Mocha, Selenium WebDriver (nbrowser tests), chai |
| **Sandbox** | Python in gVisor / Pyodide / native subprocess |
| **Package Manager** | Yarn (v1) |
| **Lint** | ESLint (flat config) |
| **CI** | GitHub Actions |
| **Container** | Docker (multi-stage build) |
| **External Storage** | S3, MinIO, Azure Blob (optional, for doc snapshots) |

## 3. Repository Shape

```
grist-core/
├── app/                    # Main application source (TypeScript)
│   ├── client/             # Browser-side code
│   │   ├── app.js          # Webpack entry point → loads App.ts
│   │   ├── ui/             # Page-level UI components (~172 files)
│   │   ├── ui2018/         # Design system tokens, icons, styled primitives
│   │   ├── components/     # Core UI components (~97 files: views, editors, grid)
│   │   ├── models/         # Client data models (DocModel, AppModel, entity records)
│   │   ├── lib/            # Client utilities (DOM helpers, autocomplete, etc.)
│   │   └── widgets/        # Cell-type widgets (TextBox, DateEditor, Choice, etc.)
│   ├── server/             # Node.js server
│   │   ├── MergedServer.ts # Main entry: combines home + docs + static servers
│   │   ├── lib/            # Server core (~175 files)
│   │   └── utils/          # Server utilities
│   ├── common/             # Shared code (client + server, ~142 files)
│   │   ├── schema.ts       # Auto-generated document schema definition
│   │   ├── DocActions.ts   # Document action types (the "action log" protocol)
│   │   ├── gristUrls.ts    # URL construction, deployment types
│   │   ├── UserAPI.ts      # API type definitions
│   │   └── ...             # ACL, Features, DocData, etc.
│   ├── gen-server/         # "General server" — home DB layer
│   │   ├── entity/         # TypeORM entities (~23: User, Org, Document, etc.)
│   │   ├── migration/      # TypeORM migrations (~50 files)
│   │   ├── lib/            # HomeDBManager, ApiServer, Housekeeper
│   │   └── ApiServer.ts    # REST API for orgs/workspaces/documents CRUD
│   └── plugin/             # Plugin API type definitions (for custom widgets)
├── sandbox/                # Python formula engine
│   └── grist/              # Core engine: engine.py, docmodel.py, etc.
├── stubs/                  # Build-flavor stubs
│   └── app/server/lib/create.ts  # CoreCreate entrypoint (OSS build)
├── test/                   # All tests
│   ├── nbrowser/           # Selenium integration tests (~188 files)
│   ├── server/             # Server unit/integration tests
│   ├── gen-server/         # Home DB tests
│   ├── client/             # Client unit tests
│   ├── common/             # Shared code tests
│   └── fixtures/           # Test documents and data
├── static/                 # Built webpack bundles + static assets
├── buildtools/             # Build scripts, webpack configs, code generators
├── plugins/                # Built-in plugins
├── docker-compose-examples/ # Docker deployment examples
├── Dockerfile              # Multi-stage production Docker build
└── _build/                 # TypeScript compilation output (tsc --build)
```

## 4. Architecture Overview

### Server Architecture

The server is built around `FlexServer` (a massive class in `app/server/lib/FlexServer.ts`) which is an Express app with incrementally-added capabilities. `MergedServer` orchestrates startup by composing server "components":

- **home**: Landing pages, org/workspace/doc CRUD API, billing, login routes
- **docs**: Document worker — loads documents, executes formulas, handles doc API
- **static**: Serves webpack bundles and static assets
- **app**: Lightweight combo for simpler deployments

### Dependency Injection via `create.ts`

Each build flavor exports a singleton `create: ICreate` from its `create.ts`:
- `CoreCreate` → OSS features, no external storage, no billing
- `EnterpriseCreate` → activation-key billing, external storage
- `SaaSCreate` → Stripe billing, per-team products, full features

The `create` object provides factory methods for: Billing, Notifier, AuditLogger, Telemetry, Assistant, Sandbox, LoginSystem, StorageManager.

### Document Lifecycle

1. `DocManager` manages active documents, maps doc IDs to `ActiveDoc` instances
2. `ActiveDoc` is the in-memory representation of an open document
3. `DocStorage` handles SQLite I/O for a single document
4. Python `engine.py` runs inside a sandbox, evaluates formulas, applies actions
5. Communication: Node ↔ Python via stdin/stdout marshaling (`NSandbox.ts`)

### Client-Server Communication

- **REST API**: Express routes in `DocApi.ts` (doc operations), `ApiServer.ts` (home operations)
- **WebSocket**: `Comm.ts` (server) ↔ `Comm.ts` (client) for real-time doc collaboration
- **Action Protocol**: All document mutations are expressed as `UserAction` arrays, sent to server, applied by Python engine, broadcast back as `DocAction` arrays

### Home Database

TypeORM-managed relational DB (SQLite default, PostgreSQL for production):
- **Entities**: User, Organization, Workspace, Document, BillingAccount, Product, AclRule, Group, Login, Secret, Config, Activation
- **HomeDBManager**: Central class for all home DB queries (access checks, org management, etc.)

## 5. Key Execution Flows

### Request Flow (REST API)
```
Browser → Express middleware (auth, org extraction, session) 
  → Route handler (ApiServer.ts or DocApi.ts) 
  → HomeDBManager (for home operations) OR ActiveDoc (for doc operations) 
  → Response
```

### Document Edit Flow
```
User types in cell → Client builds UserAction 
  → WebSocket → Comm → ActiveDoc.applyUserActions() 
  → Python sandbox processes action → returns DocActions 
  → ActiveDoc saves to SQLite, broadcasts via WebSocket 
  → All connected clients apply DocActions to their DocData
```

### Auth Flow (Core)
```
coreLogins.ts checks configured providers in order:
  1. GetGrist.com login (if configured)
  2. OIDC (if configured via env vars)
  3. SAML (if configured)
  4. Forward Auth (if configured)
  5. Minimal login (fallback — boot key or test login)
```

### Build Flow
```
buildtools/build.sh:
  1. tsc --build (TypeScript → _build/)
  2. buildtools/update_type_info.sh (generates *-ti.ts runtime type checkers)
  3. webpack (bundles client JS → static/*.bundle.js)
```

### Page Serving Flow
```
Browser requests / → AppEndpoint.ts 
  → sendAppPage() renders app.html via Handlebars 
  → Injects GristLoadConfig as JSON in page 
  → Browser loads main.bundle.js 
  → app.js creates AppImpl → sets up Comm, routing, UI
```

## 6. Important Modules

### Server Core
| File | Purpose |
|------|---------|
| `FlexServer.ts` | Central Express app, ~2000 lines, adds all routes/middleware |
| `MergedServer.ts` | Entry point, composes server types, manages startup |
| `ActiveDoc.ts` | In-memory document: formula eval, action application, sharing |
| `DocManager.ts` | Maps doc IDs → ActiveDoc, handles open/close/fork |
| `DocApi.ts` | REST API routes for document operations (CRUD rows/columns/tables) |
| `DocStorage.ts` | SQLite document persistence layer |
| `NSandbox.ts` | Spawns and communicates with Python sandbox |
| `Authorizer.ts` | Request authentication, user extraction, access checks |
| `Comm.ts` | WebSocket server for real-time collaboration |
| `AppEndpoint.ts` | Serves the main app HTML page |
| `ICreate.ts` | Build-flavor dependency injection interface |

### Client Core
| File | Purpose |
|------|---------|
| `app.js` | Webpack entry point, bootstraps AppImpl |
| `ui/App.ts` | Main app shell (AppImpl class) |
| `models/AppModel.ts` | Top-level app state (current org, user, etc.) |
| `models/DocModel.ts` | Observable models for all document metadata tables |
| `models/DocPageModel.ts` | State for the current document page |
| `components/GristDoc.ts` | Main document view controller |
| `components/Comm.ts` | WebSocket client for server communication |
| `components/BaseView.ts` | Base class for grid/card/chart views |
| `ui2018/cssVars.ts` | Design tokens (colors, spacing, typography) |

### Home Database
| File | Purpose |
|------|---------|
| `gen-server/lib/homedb/HomeDBManager.ts` | All home DB queries (enormous file, ~5000+ lines) |
| `gen-server/ApiServer.ts` | REST routes for orgs, workspaces, docs |
| `gen-server/entity/*.ts` | TypeORM entity definitions |
| `gen-server/migration/*.ts` | Database migrations (numbered by timestamp) |

### Python Sandbox
| File | Purpose |
|------|---------|
| `sandbox/grist/engine.py` | Formula evaluation engine |
| `sandbox/grist/docmodel.py` | Python-side document model |
| `sandbox/grist/main.py` | Sandbox entry point, exposes methods to Node |
| `sandbox/grist/schema.py` | Python-side schema definition |
| `sandbox/grist/actions.py` | Action types and application |

### Shared
| File | Purpose |
|------|---------|
| `common/schema.ts` | Auto-generated meta-table schema (DO NOT EDIT MANUALLY) |
| `common/DocActions.ts` | Action type definitions used everywhere |
| `common/UserAPI.ts` | API type definitions |
| `common/gristUrls.ts` | URL routing, deployment types, feature flags |
| `common/Features.ts` | Feature definitions per product tier |
| `common/ACLRuleCollection.ts` | Access rule parsing and evaluation |

## 7. Conventions

### Naming
- Files: PascalCase for classes/components (`ActiveDoc.ts`), camelCase for utilities (`requestUtils.ts`)
- CSS-in-JS: Styled functions exported from `*Css.ts` files (using grainjs `styled()`)
- Interfaces: `I`-prefix for dependency-injection interfaces (`ICreate`, `ISandbox`, `INotifier`)
- Test files: named after the feature they test (e.g., `test/nbrowser/AccessRules1.ts`)

### Patterns
- **grainjs observables**: UI uses `Observable`, `Computed`, `dom.domComputed()`, `dom.maybe()` for reactive rendering
- **Knockout observables**: Legacy code still uses `ko.observable()` — mixed with grainjs via `fromKo()`/`toKo()`
- **Disposable pattern**: Components extend `Disposable`, use `autoDispose()` for cleanup
- **Action-based mutations**: ALL document changes go through the action system — never modify SQLite directly
- **Runtime type checking**: `-ti.ts` files (auto-generated by `ts-interface-builder`) provide runtime validators
- **Localization**: `makeT("ModuleName")` for translations, keys auto-extracted by build script

### Style
- 2-space indentation
- No semicolons required (but widely used)
- ESLint enforced with `--max-warnings=0`
- Prefer `import` over `require` (some legacy `.js` files still use `require`)

### Architecture Rules
- `app/common/` must not import from `app/client/` or `app/server/`
- `app/client/` must not import from `app/server/`
- `app/gen-server/` can import from `app/common/` only
- The `create.ts` DI pattern means build-specific code lives in `stubs/` (core) or `ext/` (enterprise) directories

## 8. Common Tasks

### Add REST API Endpoint (Document-level)
1. Define the route in `app/server/lib/DocApi.ts` using `app.get/post/patch/delete()`
2. Add type definitions in `app/common/ActiveDocAPI.ts` or `app/plugin/DocApiTypes.ts`
3. If new types, generate runtime checkers: `buildtools/update_type_info.sh app`
4. Add client-side API method in `app/common/UserAPI.ts` → `DocAPIImpl` class
5. Write test in `test/server/DocApi.ts` or `test/nbrowser/`

### Add REST API Endpoint (Home-level)
1. Add route in `app/gen-server/ApiServer.ts`
2. Add query/mutation in `app/gen-server/lib/homedb/HomeDBManager.ts`
3. Add client API in `app/common/UserAPI.ts` → `UserAPIImpl` class
4. Write test in `test/gen-server/`

### Add UI Page
1. Create page component in `app/client/ui/MyPage.ts` using grainjs
2. Create CSS file `app/client/ui/MyPageCss.ts` using `styled()` pattern
3. Add route in `app/client/ui/AppUI.ts` or `app/client/models/AppModel.ts`
4. If it needs a server endpoint, add in `FlexServer.ts` (`addMyPage()` method)
5. Add `sendAppPage()` call in `AppEndpoint.ts` or `FlexServer.ts`

### Add Database Column/Table (Home DB)
1. Modify the entity in `app/gen-server/entity/`
2. Create a new migration: `app/gen-server/migration/<timestamp>-<Name>.ts`
3. Update `HomeDBManager.ts` with new queries
4. Build with `yarn tsc --build`

### Add Document Schema Field (Meta-table)
1. Edit `sandbox/grist/schema.py` (Python source of truth)
2. Run `npm run generate:schema:ts` to regenerate `app/common/schema.ts`
3. Add migration in `sandbox/grist/migrations.py`
4. Update `DocModel.ts` entity record if needed on client
5. Build and test

### Write an nbrowser Test
1. Create file in `test/nbrowser/MyTest.ts`
2. Use `setupTestSuite()`, `session.tempNewDoc()` from `gristUtils`
3. Use Selenium WebDriver API via `driver.find()`, `driver.sendKeys()`
4. Run: `GREP_TESTS="MyTest" npm run test:nbrowser`
5. Keep wait timeouts to 1s max on localhost

### Debug Production Issue
1. Check server logs (structured JSON via `app/server/lib/log.ts`)
2. Use boot page (`/boot`) with boot key for diagnostics
3. Check `_grist_DocInfo` table in the SQLite document
4. Use testing hooks if available (`--testingHooks` flag)

## 9. Commands

```bash
# Development
yarn install                    # Install dependencies
npm run start                   # Start dev server with file watching (sandbox/watch.sh)
npm run build                   # Build: tsc + webpack (development mode)
npm run build:prod              # Build: tsc + webpack (production mode)

# Type checking
yarn tsc --build                # TypeScript type check (preferred over tsc --noEmit)

# Testing
GREP_TESTS="MyTest" npm test    # Run specific tests across all suites
npm run test:nbrowser           # Run all nbrowser (Selenium) tests
npm run test:server             # Run server tests
npm run test:gen-server         # Run home DB tests
npm run test:client             # Run client unit tests
npm run test:common             # Run common module tests
npm run test:python             # Run Python sandbox tests
GREP_TESTS="MyTest" npm run test:nbrowser  # Run specific nbrowser test

# Lint
npm run lint                    # ESLint check
npm run lint:fix                # ESLint auto-fix

# Code generation
npm run generate:schema:ts      # Regenerate schema.ts from Python
npm run generate:icons          # Regenerate icon CSS from SVGs
npm run generate:translation    # Extract translation keys

# Docker
docker build -t grist .         # Build Docker image

# Storybook
npm run storybook               # Start Storybook dev server on :6006
```

## 10. Sharp Edges / Risk Areas

### Dangerous Files
- **`FlexServer.ts`** (~2000 lines): Monolithic server setup. Changes here affect everything. Methods must be called in specific order during startup.
- **`HomeDBManager.ts`** (~5000+ lines): All home DB logic. Complex access-check queries. Easy to introduce permission bugs.
- **`ActiveDoc.ts`**: In-memory doc state with complex concurrency. Race conditions possible.
- **`schema.ts`**: AUTO-GENERATED — do not edit manually. Edit `sandbox/grist/schema.py` and regenerate.
- **`*-ti.ts` files**: Auto-generated runtime type checkers. Regenerate via `buildtools/update_type_info.sh`.

### Architectural Risks
- **Action system complexity**: All doc mutations must go through UserAction → Python sandbox → DocAction pipeline. Bypassing this breaks undo/redo, access control, and replication.
- **Python sandbox security**: Formula execution is sandboxed (gVisor/Pyodide). Changes to sandbox configuration can create security holes.
- **Three build flavors**: Code must work in core, enterprise, and SaaS builds. The `create.ts` DI pattern means some code paths are only tested in specific builds.
- **Mixed observable systems**: grainjs and Knockout coexist. Interop via `fromKo()`/`toKo()` can create subtle disposal and update-ordering bugs.
- **WebSocket state**: Real-time collaboration relies on ordered action delivery. Network issues can cause state divergence.

### Testing Risks
- **nbrowser tests are flaky**: Selenium + async UI = timing issues. Use `waitForServer()`, avoid brittle selectors, keep waits to 1s max.
- **Test server reuse**: Many nbrowser tests share a server instance. State leakage between tests is possible.
- **No hot-reload for tests**: Must rebuild (`npm run build`) before running tests against changed code.

## 11. Fastest Way To Become Productive

### Reading Order
1. **This file** — you're here
2. **`app/server/MergedServer.ts`** — understand server startup and component composition
3. **`app/server/lib/ICreate.ts`** — understand the build-flavor DI pattern
4. **`app/common/DocActions.ts`** — understand the action protocol (fundamental to everything)
5. **`app/server/lib/ActiveDoc.ts`** (first 100 lines) — understand document lifecycle
6. **`app/server/lib/DocApi.ts`** (first 100 lines) — understand REST API pattern
7. **`app/client/app.js`** + **`app/client/ui/App.ts`** — understand client bootstrap
8. **`app/client/models/DocModel.ts`** — understand client-side data model
9. **`test/nbrowser/gristUtils.ts`** (first 100 lines) — understand test patterns

### First Actions
1. Run `npm run start` and open in browser — see the app working
2. Make a trivial UI change (e.g., a label in `app/client/ui/`) and see it hot-reload
3. Run a single nbrowser test: `GREP_TESTS="Smoke" npm run test:nbrowser`
4. Read through one complete API endpoint in `DocApi.ts` end-to-end
5. Open a `.grist` file with an SQLite viewer to see the document structure

## 12. Suggested Next Investigations

- **`ext/` directory structure**: How enterprise extensions overlay core code (not present in core-only checkout)
- **`sandbox/grist/engine.py`**: Deep dive into the Python formula engine (dependency tracking, recalculation)
- **`app/common/ACLRuleCollection.ts`** + **`app/server/lib/GranularAccess.ts`**: Access control rule system
- **`app/server/lib/HostedStorageManager.ts`**: How document snapshots work with external storage
- **`app/client/components/GristDoc.ts`**: Main document UI controller — ties together views, sidebar, formula bar
- **Webhook/trigger system**: `app/server/lib/DocApiTriggers.ts` and `app/server/lib/Triggers.ts`
- **Plugin/widget system**: `app/plugin/` API + `app/server/lib/PluginManager.ts`
- **Migration patterns**: Both TypeORM migrations (home DB) and Python migrations (document schema)
