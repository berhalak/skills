---
name: grist-nbrowser-tests
description: Use when writing, reading, or debugging nbrowser integration tests in the Grist codebase — Mocha + Selenium WebDriver tests in test/nbrowser/, using gristUtils, setupTestSuite, session management, cell interaction, or browser automation for Grist.
---

# Grist nbrowser Tests

Integration tests using **Mocha + mocha-webdriver + Selenium**. Tests run against a real Grist server with a test database and a real browser.

---

## Test File Template

```typescript
import * as gu from "test/nbrowser/gristUtils";
import { server, setupTestSuite } from "test/nbrowser/testUtils";
import { assert, driver, Key } from "mocha-webdriver";

describe("MyFeature", function() {
  this.timeout(20000);
  const cleanup = setupTestSuite();

  afterEach(() => gu.checkForErrors());

  it("should do something", async function() {
    const session = await gu.session().login();
    const docId = await session.tempNewDoc(cleanup, "TestDoc");
    await session.loadDoc(`/doc/${docId}`);

    // Interact
    await gu.getCell("A", 1).click();
    await gu.enterCell("hello");
    await gu.waitForServer();

    // Assert
    assert.equal(await gu.getCell("A", 1).getText(), "hello");
  });
});
```

---

## Key Patterns

| Pattern | How |
|---|---|
| **Setup** | `const cleanup = setupTestSuite()` at `describe` level |
| **Login** | `gu.session().login()` or `server.simulateLogin(name, email, org)` |
| **Create temp doc** | `session.tempNewDoc(cleanup, "Name")` — auto-cleaned |
| **Load doc** | `session.loadDoc("/doc/" + docId)` |
| **Load from fixture** | `session.tempDoc(cleanup, "FixtureName.grist")` |
| **Cell interaction** | `gu.getCell(col, row)`, `gu.enterCell(value)`, `gu.sendKeys(...)` |
| **Wait for save** | `gu.waitForServer()` after data-changing actions |
| **Error checking** | `gu.checkForErrors()` in `afterEach` |
| **Bulk data setup** | `session.createHomeApi().applyUserActions(docId, actions)` via API |
| **Flaky waits** | `gu.waitToPass(async () => { ... })` for eventually-consistent checks |
| **Undo** | `gu.undo()` or `gu.undo(count)` |

---

## gristUtils Quick Reference (`gu`)

**Session & Navigation:**
- `gu.session()` — returns `Session.default` (fluent builder)
- `session.user("user1")` — switch user
- `session.teamSite`, `session.personalSite` — predefined sites
- `session.login()`, `session.loadDoc(path)`, `session.loadDocMenu(path)`
- `session.createHomeApi()` — get API client for direct data manipulation

**Grid Cells:**
- `gu.getCell(col, row)` — get cell element (col is name or 0-based index)
- `gu.getVisibleGridCells({col, rowNums})` — read multiple cell values
- `gu.enterCell(value)` — type into active cell
- `gu.getColumnNames()` — list column headers
- `gu.getGridRowCount()` — count visible rows

**UI Interaction:**
- `gu.sendKeys(...keys)` — keyboard input
- `gu.openColumnMenu(col, option?)` — open column context menu
- `gu.selectSectionByTitle(title)` — switch active widget/section
- `gu.addNewSection(type, table)` — add a new widget
- `gu.toggleSidePanel("right" | "left")` — open/close panels
- `gu.bigScreen()` / `gu.narrowScreen()` — resize browser window

**Data Actions (via API):**
- `gu.sendActions(actions)` — send user actions to the document
- `gu.sendCommand(name, args)` — send a command

**Waiting & Assertions:**
- `gu.waitForServer()` — wait for pending requests to complete
- `gu.waitForDocToLoad()` — wait for document to fully load
- `gu.waitToPass(fn, timeoutMs?)` — retry `fn` until it passes
- `gu.checkForErrors()` — assert no unexpected console errors

---

## Test Users

Predefined test users: `"user1"`, `"user2"`, `"user3"`, `"chimpy"`, `"kiwi"`, `"anon"`.

```typescript
// Default session (chimpy)
const session = await gu.session().login();

// Specific user
const session = await gu.session().user("user1").login();

// Team site
const session = await gu.session().teamSite.login();
```

---

## Data Setup Patterns

**Empty doc with columns:**
```typescript
const docId = await session.tempNewDoc(cleanup, "Test");
const api = session.createHomeApi();
await api.applyUserActions(docId, [
  ["AddTable", "People", [{id: "Name"}, {id: "Age", type: "Int"}]],
  ["BulkAddRecord", "People", [null, null], {Name: ["Alice", "Bob"], Age: [30, 25]}],
]);
await session.loadDoc(`/doc/${docId}`);
```

**From fixture:**
```typescript
const docId = await session.tempDoc(cleanup, "World.grist");
await session.loadDoc(`/doc/${docId}`);
```

---

## Running Tests

```bash
# All nbrowser tests
./test/testrun.sh nbrowser

# Filter by test name
GREP_TESTS="MyFeature" ./test/testrun.sh nbrowser

# Debug mode (REPL on failure, screenshots)
DEBUG=1 ./test/testrun.sh nbrowser

# Verbose server output
VERBOSE=1 ./test/testrun.sh nbrowser
```

---

## Server Lifecycle

`setupTestSuite()` handles everything:
1. Spawns real Grist server (`devServerMain`) with test SQLite DB
2. Assigns unique ports, connects testing hooks via Unix socket
3. After each suite: logs out, clears storage, closes extra windows
4. Screenshots and server logs captured on failure

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Forgetting `waitForServer()` after data change | Always await it after clicks/edits that modify data |
| Not using `cleanup` with `tempNewDoc` | Pass `cleanup` so docs are auto-removed |
| Missing `afterEach(() => gu.checkForErrors())` | Always include — catches silent JS errors |
| Hardcoding cell positions after row changes | Use `gu.getVisibleGridCells` to read actual state |
| Not waiting for UI transitions | Use `gu.waitToPass()` or `driver.findWait(selector, ms)` |

---

## File Locations

- Tests: `test/nbrowser/*.ts`
- Core test utils: `core/test/nbrowser/gristUtils.ts`, `testUtils.ts`
- WebDriver utils: `core/test/nbrowser/gristWebDriverUtils.ts`
- Home/login utils: `core/test/nbrowser/homeUtil.ts`
- Test fixtures: `test/fixtures/`
- Mocha config: `core/test/init-mocha-webdriver.js`
- Test runner: `test/testrun.sh`
