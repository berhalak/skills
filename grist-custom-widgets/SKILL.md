---
name: grist-custom-widgets
description: Reference for building Grist custom widgets. Use when creating or modifying custom widgets, working with grist-plugin-api, table manipulation via applyUserActions, conditional formatting, trigger formulas, view sections, or widget theming.
---

# Grist Custom Widget Development

## Project Structure

Each widget lives in its own directory under the grist-widget repo:

```
widgetname/
  index.html    # Single HTML file (or with separate JS/CSS)
  package.json  # For auto-build/publish
```

### package.json

```json
{
  "name": "@gristlabs/widget-widgetname",
  "description": "Widget Name",
  "homePage": "https://github.com/gristlabs/grist-widget",
  "version": "0.0.1",
  "grist": {
    "name": "Widget Name",
    "url": "https://gristlabs.github.io/grist-widget/widgetname/index.html",
    "widgetId": "@gristlabs/widget-widgetname",
    "published": true,
    "accessLevel": "full",
    "renderAfterReady": true,
    "description": "Short description.",
    "authors": [{ "name": "Author", "url": "https://github.com/author" }],
    "isGristLabsMaintained": false
  }
}
```

### manifest.json

Add an entry to the root `manifest.json` with the same fields as `grist` in package.json, plus `"lastUpdatedAt"`. The `url` uses `localhost:8585` for dev.

## Widget HTML Boilerplate

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Widget Name</title>
  <script src="https://docs.getgrist.com/grist-plugin-api.js"></script>
</head>
<body>
  <script>
    grist.ready({ requiredAccess: 'full' });

    let tableId = null;
    grist.on('message', (e) => {
      if (e.tableId) { tableId = e.tableId; }
    });

    grist.onRecords((recs) => {
      // recs is array of row-oriented objects with all columns
      // Only includes columns that have fields in the widget's view section
    });
  </script>
</body>
</html>
```

## Core API

### Reading Data

- `grist.onRecords(callback)` — called when table data changes. Returns row-oriented objects. **Only includes columns that have fields in the widget's view section** (see "View Section Fields" below).
- `grist.docApi.fetchTable(tableId)` — returns column-oriented data: `{ id: [...], colName: [...], ... }`. Always returns all columns.
- `grist.docApi.fetchTable('_grist_Tables')` — read metadata tables.

### Writing Data — applyUserActions

`grist.docApi.applyUserActions(actions)` takes an array of actions. **Always batch multiple actions into a single call** for performance.

```javascript
await grist.docApi.applyUserActions([
  ['AddRecord', tableId, null, { Col1: 'value' }],
  ['UpdateRecord', tableId, rowId, { Col1: 'new value' }],
  ['BulkRemoveRecord', tableId, [rowId1, rowId2]],
]);
```

### Column Actions

- **`AddVisibleColumn`** (NOT `AddColumn`) — creates a column AND adds it to the view section so it's visible:
  ```javascript
  ['AddVisibleColumn', tableId, 'ColName', { type: 'Text', isFormula: false }]
  ```
- `RemoveColumn`: `['RemoveColumn', tableId, 'ColName']`
- `ModifyColumn`: `['ModifyColumn', tableId, 'ColName', { type: 'Numeric' }]`

**Important:** `AddColumn` creates the column but does NOT add it to any view section. Always use `AddVisibleColumn` unless you specifically want a hidden column.

### Getting Table Reference

```javascript
async function getTableRef() {
  const tables = await grist.docApi.fetchTable('_grist_Tables');
  for (let i = 0; i < tables.tableId.length; i++) {
    if (tables.tableId[i] === tableId) return tables.id[i];
  }
  return null;
}
```

## Trigger Formulas

Trigger formulas run on data columns when specific conditions are met. Set via `UpdateRecord` on `_grist_Tables_column`:

```javascript
['UpdateRecord', '_grist_Tables_column', colRef, {
  formula: 'user.Name if value == "x" else value',
  recalcWhen: 0,           // RecalcWhen.DEFAULT
  recalcDeps: ['L', colRef] // RefList pointing to itself
}]
```

### RecalcWhen Values

| Value | Name | Behavior |
|-------|------|----------|
| `0` | DEFAULT | Recalc on new records OR when fields in `recalcDeps` change. If `recalcDeps` includes itself = **"Current field (data cleaning)"** mode |
| `1` | NEVER | Never auto-recalculate |
| `2` | MANUAL_UPDATES | Recalc on new records and on manual updates to **any** data field |

### recalcDeps Format

RefList format: `['L', colRef1, colRef2, ...]` where each ref is a column ID from `_grist_Tables_column`.

## Conditional Formatting

Three-step process:

### 1. Add Empty Rule
```javascript
['AddEmptyRule', tableId, 0, colRef]
// Args: tableId, fieldRef (0 for column-level), colRef
// Creates a hidden gristHelper_ConditionalRule column
```

### 2. Find the Rule Column ID
```javascript
const updatedCols = await grist.docApi.fetchTable('_grist_Tables_column');
for (let i = 0; i < updatedCols.id.length; i++) {
  if (updatedCols.id[i] === colRef) {
    const rules = updatedCols.rules[i];
    // RefList format: ['L', ruleColId1, ...]
    if (rules && Array.isArray(rules) && rules.length > 1) {
      ruleColId = rules[rules.length - 1]; // last added rule
    }
    break;
  }
}
```

### 3. Set Formula and Style
```javascript
await grist.docApi.applyUserActions([
  ['UpdateRecord', '_grist_Tables_column', ruleColId, {
    formula: '$Active == "C1"'
  }],
  ['UpdateRecord', '_grist_Tables_column', colRef, {
    widgetOptions: JSON.stringify({
      widget: 'TextBox',
      alignment: 'center',
      rulesOptions: [{ fillColor: '#FF0000', textColor: '#FFFFFF', fontBold: true }]
    })
  }]
]);
```

### Style Interface
```javascript
{
  textColor: '#FF0000',      // optional
  fillColor: '#00FF00',      // optional
  fontBold: true,             // optional
  fontItalic: false,          // optional
  fontUnderline: false,       // optional
  fontStrikethrough: false    // optional
}
```

`rulesOptions` array must match the number of rules in the `rules` RefList (one style per rule, in order).

**Note:** `gristHelper_ConditionalRule` columns are auto-deleted when their parent column is removed. When removing columns, skip `gristHelper_*` to avoid errors.

## View Section Fields

### Why It Matters

`grist.onRecords()` only returns columns that have fields in the widget's view section. If you create new columns, you must add them to the widget's section for `onRecords` to include them.

### Adding Fields to Widget Section

```javascript
// Find the custom widget's view section
const sections = await grist.docApi.fetchTable('_grist_Views_section');
let widgetSectionRef = null;
for (let i = 0; i < sections.id.length; i++) {
  if (sections.tableRef[i] === tableRef) {
    const kind = String(sections.parentKey[i] || '');
    if (kind === 'custom') {
      widgetSectionRef = sections.id[i];
      break;
    }
  }
}

// Add field records for each column
await grist.docApi.applyUserActions([
  ['AddRecord', '_grist_Views_section_field', null, {
    parentId: widgetSectionRef,
    colRef: colRef,
  }]
]);
```

### parentKey Values for View Sections
- `'record'` — grid/table view
- `'custom'` — custom widget view
- `'detail'` — detail/card view
- `'chart'` — chart view

### Setting Column Widths

Update `width` on `_grist_Views_section_field` records:

```javascript
const fields = await grist.docApi.fetchTable('_grist_Views_section_field');
const actions = [];
for (let i = 0; i < fields.id.length; i++) {
  if (fields.parentId[i] === sectionRef) {
    actions.push(['UpdateRecord', '_grist_Views_section_field', fields.id[i], { width: 50 }]);
  }
}
await grist.docApi.applyUserActions(actions);
```

## Metadata Tables Reference

| Table | Key Columns | Purpose |
|-------|-------------|---------|
| `_grist_Tables` | `tableId`, `id` | Table name to ref mapping |
| `_grist_Tables_column` | `colId`, `parentId`, `type`, `formula`, `recalcWhen`, `recalcDeps`, `rules`, `widgetOptions` | Column metadata |
| `_grist_Views_section` | `tableRef`, `parentKey` | View sections |
| `_grist_Views_section_field` | `parentId`, `colRef`, `width`, `widgetOptions` | Fields in view sections |

## Theming

Use Grist's CSS variables for theme compatibility (light + dark):

```css
body {
  background: var(--grist-theme-page-panels-main-panel-bg, inherit);
  color: var(--grist-theme-text, inherit);
}
h1 { color: var(--grist-theme-cursor, #16b378); }
.muted { color: var(--grist-theme-text-light, #929299); }
button {
  background: var(--grist-theme-cursor, #16b378);
  color: #fff;
}
```

For backgrounds that must work in both themes, prefer `rgba()`:
```css
.card { background: rgba(128, 128, 128, 0.08); }
.highlight { background: rgba(22, 179, 120, 0.12); }
```

### Key CSS Variables
- `--grist-theme-cursor` — primary green (#16b378 light / #1da270 dark)
- `--grist-theme-text` — body text
- `--grist-theme-text-light` — secondary text
- `--grist-theme-page-panels-main-panel-bg` — main background

## Multiplayer / Stateless Design

Widgets should derive ALL state from the table data, not local variables. Multiple users may have the widget open simultaneously.

### Race Condition Prevention

When multiple widgets may try to write simultaneously:

```javascript
// 1. Random delay to stagger
await new Promise(r => setTimeout(r, Math.floor(Math.random() * 300)));

// 2. Re-fetch and verify before writing
const freshData = await grist.docApi.fetchTable(tableId);
const currentValue = String(freshData.SomeCol[rowIndex] || '').trim();
if (currentValue !== expectedValue) {
  // Someone else already acted — skip
  return;
}

// 3. Proceed with write
await grist.docApi.applyUserActions([...]);
```

### Avoiding Flicker

Use a `busy` flag to skip `onRecords` processing while writing to the table, so intermediate states don't cause UI flicker:

```javascript
let busy = false;

grist.onRecords((recs) => {
  if (busy) return; // Skip intermediate states
  // ... process records
});

async function doWrite() {
  busy = true;
  try {
    await grist.docApi.applyUserActions([...]);
  } finally {
    busy = false;
  }
}
```

## Column Removal Best Practices

When removing all columns from a table:
- Skip `manualSort` and `id` (internal columns)
- Skip `gristHelper_*` (auto-deleted with parent columns)

```javascript
const colMeta = await grist.docApi.fetchTable('_grist_Tables_column');
const removeActions = [];
for (let i = 0; i < colMeta.id.length; i++) {
  if (colMeta.parentId[i] === tableRef) {
    const cid = colMeta.colId[i];
    if (cid.startsWith('manualSort') || cid === 'id' || cid.startsWith('gristHelper_')) continue;
    removeActions.push(['RemoveColumn', tableId, cid]);
  }
}
await grist.docApi.applyUserActions(removeActions);
```
