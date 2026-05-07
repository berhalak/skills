# Grist Action System & Python Engine

This document covers Grist's action protocol: the two action layers (UserActions vs DocActions), how to apply actions via the REST API, every UserAction the Python engine accepts, and the shape of the response the engine returns.

## 1. Two-Layer Action Model

Every document mutation flows through this pipeline. **There is no other supported way to change a document** — bypassing it breaks undo/redo, ACL, replication, and real-time sync.

```
Client/API → UserAction[] → ActiveDoc.applyUserActions()
                          → Python sandbox (engine.py)
                          → DocAction[]  (the low-level "what actually happened")
                          → SQLite write + WebSocket broadcast → all connected clients
```

### UserAction (high-level, intent)

A `UserAction` is `[ActionName, ...args]`. It expresses what the *user* wants to do — e.g. "add an empty table named Foo", "upsert a row keyed on `email`". User actions can be composite: a single `AddEmptyTable` action expands inside the engine into many DocActions (creating the table, creating columns, creating a primary view, creating a view section, adding fields, …).

UserActions are defined and implemented in `sandbox/grist/useractions.py` (each is decorated with `@useraction`). The TS type is `UserAction = (string | number | object | boolean | null | undefined)[]` (`app/common/DocActions.ts:193`).

### DocAction (low-level, primitive)

A `DocAction` is also `[ActionName, ...args]` but is restricted to ~14 primitive operations on tables, columns, and rows. DocActions are what get persisted to the action log, replayed on undo, and broadcast to other clients. They are defined exactly in `app/common/DocActions.ts` and `sandbox/grist/actions.py`.

There are 14 DocActions, split into **schema actions** (table/column structure) and **data actions** (rows):

| DocAction | Tuple shape | Notes |
|---|---|---|
| `AddRecord` | `["AddRecord", tableId, rowId, colValues]` | Single row insert. `rowId` may be `null` to auto-assign. |
| `BulkAddRecord` | `["BulkAddRecord", tableId, rowIds[], bulkColValues]` | Column-oriented bulk insert. |
| `RemoveRecord` | `["RemoveRecord", tableId, rowId]` | |
| `BulkRemoveRecord` | `["BulkRemoveRecord", tableId, rowIds[]]` | |
| `UpdateRecord` | `["UpdateRecord", tableId, rowId, colValues]` | |
| `BulkUpdateRecord` | `["BulkUpdateRecord", tableId, rowIds[], bulkColValues]` | |
| `ReplaceTableData` | `["ReplaceTableData", tableId, rowIds[], bulkColValues]` | Wipe & reload. |
| `TableData` | `["TableData", tableId, rowIds[], bulkColValues]` | Used in fetches, not normally as input. |
| `AddTable` | `["AddTable", tableId, colInfoWithId[]]` | Schema action. |
| `RemoveTable` | `["RemoveTable", tableId]` | Schema action. |
| `RenameTable` | `["RenameTable", oldId, newId]` | Schema action. |
| `AddColumn` | `["AddColumn", tableId, colId, colInfo]` | Schema action. |
| `RemoveColumn` | `["RemoveColumn", tableId, colId]` | Schema action. |
| `RenameColumn` | `["RenameColumn", tableId, oldColId, newColId]` | Schema action. |
| `ModifyColumn` | `["ModifyColumn", tableId, colId, partialColInfo]` | Schema action. |

`ColInfo` is `{ type: string, isFormula: boolean, formula: string }`. `ColValues` is `{ [colId: string]: CellValue }`. `BulkColValues` is `{ [colId: string]: CellValue[] }` (column-oriented).

You **can** submit DocActions directly as UserActions — `ApplyDocActions` (defined in `useractions.py`) is itself a UserAction that takes a list of DocActions and applies them verbatim. But the higher-level UserActions are usually what you want, since they keep meta-tables, views, formulas, and reverse references in sync.

## 2. Applying Actions From the API

### Endpoint

```
POST /api/docs/:docId/apply
Content-Type: application/json

[
  ["AddEmptyTable", null],
  ["AddRecord", "Table1", null, {"A": 1, "B": "hello"}]
]
```

- Body is a JSON array of UserActions (each itself an array `[name, ...args]`).
- Auth: requires editor permission on the doc (`canEdit` middleware).
- Query param `?noparse=1` disables string-to-typed-value coercion (e.g. won't parse `"2024-01-01"` into a Date for a Date column).
- Defined at `app/server/lib/DocApi.ts:190` — handler simply forwards the body to `ActiveDoc.applyUserActions()`.

### Response shape — `ApplyUAResult`

Defined at `app/common/ActiveDocAPI.ts:30`:

```ts
{
  actionNum: number,         // sequence number assigned to this bundle in the action log
  actionHash: string | null, // hash of the recorded action (null if nothing was recorded)
  retValues: any[],          // ONE entry per UserAction submitted, in order
  isModification: boolean    // true if the document was actually modified
}
```

The interesting field is `retValues`: it's an array the same length as the input, where each element is whatever the corresponding UserAction's Python implementation returned (see action catalog below).

Example: posting `[["AddEmptyTable", null], ["AddRecord", "Table1", null, {"A": 1}]]` returns `retValues` like:

```json
[
  { "id": 1, "table_id": "Table1", "columns": ["A", "B", "C"], "views": [{"id": 1, "sections": [2]}] },
  1
]
```

### Other doc-level endpoints that internally apply actions

Most of `DocApi.ts` is sugar over `applyUserActions`:

- `POST /api/docs/:docId/tables/:tableId/data` → `BulkAddRecord`
- `PATCH /api/docs/:docId/tables/:tableId/data` → `BulkUpdateRecord`
- `POST /api/docs/:docId/tables/:tableId/data/delete` → `BulkRemoveRecord`
- `POST /api/docs/:docId/tables/:tableId/records` → `BulkAddRecord` (records-format input)
- `PUT /api/docs/:docId/tables/:tableId/records` → `BulkAddOrUpdateRecord` (upsert)
- `POST /api/docs/:docId/tables` → `AddTable`
- `PATCH /api/docs/:docId/tables` → updates `_grist_Tables` rows
- `POST /api/docs/:docId/tables/:tableId/columns` → adds rows to `_grist_Tables_column`

Use `/apply` directly when you need to send multiple actions atomically (one bundle, one undo entry, one ACL check pass) or when you need a UserAction that has no dedicated REST sugar.

### Programmatic entry points (server code, not REST)

- `ActiveDoc.applyUserActions(docSession, actions, options)` — `app/server/lib/ActiveDoc.ts:1615`. Public method, used by `DocApi.ts`, the WebSocket handler in `Comm.ts`, and internal callers.
- `ActiveDoc.applyUserActionsById(docSession, actionNums, actionHashes, undo, options)` — apply or undo actions referenced by id from the action history.
- `ActiveDoc._applyUserActionsAsSystem(actions)` — `app/server/lib/ActiveDoc.ts:2486`. Used by housekeeping (`Calculate`, `UpdateCurrentTime`, `RemoveStaleObjects`).

## 3. Engine Internals — How a UserAction Becomes DocActions

Inside the sandbox (`sandbox/grist/engine.py:1297`):

```python
def apply_user_actions(self, user_actions, user=None):
    self.out_actions = action_obj.ActionGroup()
    for ua in user_actions:
        self.out_actions.retValues.append(self._apply_one_user_action(ua))
    self.out_actions.flush_calc_changes()
    self.out_actions.check_sanity()
    return self.out_actions
```

`ActionGroup` (`sandbox/grist/action_obj.py`) is the unit returned to Node:

```python
class ActionGroup:
    calc      # DocActions from formula recalculation
    stored    # DocActions to persist to the action log
    direct    # parallel bool[] — was each `stored` action directly requested?
    undo      # inverse DocActions for undo
    retValues # one entry per input UserAction (this is what reaches the API caller)
    summary   # ActionSummary for incremental diffs
    requests  # for formulas that use REQUEST()
```

The Node side (`ActiveDoc`) takes the `ActionGroup`, persists `stored` to SQLite + the action history, broadcasts the bundle to other clients via `Comm`, and returns `retValues` (plus `actionNum`/`actionHash`/`isModification`) to the API caller as `ApplyUAResult`.

## 4. UserAction Catalog

All defined in `sandbox/grist/useractions.py`. Each row shows the action shape as you'd send it (`["Name", arg1, arg2, ...]`) and what the engine returns in `retValues[i]` for that action.

### Maintenance / housekeeping

These are usually applied automatically (set `SYSTEM_ACTIONS` in `app/common/DocActions.ts:201`):

| Action | Returns |
|---|---|
| `["InitNewDoc"]` | `None` — applied once on doc creation |
| `["Calculate"]` | `None` — triggers formula recalc |
| `["UpdateCurrentTime"]` | `None` — recalculates time-dependent formulas |
| `["RespondToRequests", responses, cachedKeys]` | `None` — supplies data for `REQUEST()` formula calls |
| `["RemoveStaleObjects"]` | `None` — cleanup of helper columns/tables at shutdown |
| `["ApplyDocActions", docActions[]]` | `None` — apply a list of raw DocActions |
| `["ApplyUndoActions", undoActions[]]` | `None` — apply undo bundle |

### Records (rows)

| Action | Returns |
|---|---|
| `["AddRecord", tableId, rowId\|null, colValues]` | The new `rowId: number` (auto-assigned if `null` was passed) |
| `["BulkAddRecord", tableId, rowIds[]\|nulls, bulkColValues]` | `number[]` — the assigned row ids in order |
| `["UpdateRecord", tableId, rowId, colValues]` | `None` |
| `["BulkUpdateRecord", tableId, rowIds[], bulkColValues]` | `None` |
| `["RemoveRecord", tableId, rowId]` | `None` |
| `["BulkRemoveRecord", tableId, rowIds[]]` | `None` |
| `["ReplaceTableData", tableId, rowIds[], bulkColValues]` | `None` (replaces table contents) |
| `["AddOrUpdateRecord", tableId, require, colValues, options]` | `{recordIds: number[], action: "ADD"\|"UPDATE"\|"NONE"}` |
| `["BulkAddOrUpdateRecord", tableId, require, colValues, options]` | `{recordIds: number[][], addRecordIds: number[], updateRecordIds: number[][]}` |

`require` and `colValues` for the upsert variants are dicts of `{colId: value}` (or `{colId: [values]}` for the bulk form). `options` accepts `{update?: bool, add?: bool, on_many?: "first"\|"none"\|"all", allow_empty_require?: bool}`.

### Columns

| Action | Returns |
|---|---|
| `["AddColumn", tableId, colId\|null, colInfo]` | `{colRef: number, colId: string}` — `colRef` is the row id in `_grist_Tables_column`, `colId` is the final (possibly de-duplicated) column identifier |
| `["AddVisibleColumn", tableId, colId\|null, colInfo]` | Same as `AddColumn`, plus adds the column as a field to all 'record' view sections |
| `["AddHiddenColumn", tableId, colId\|null, colInfo]` | Same as `AddColumn`, but no view-field is created |
| `["RemoveColumn", tableId, colId]` | `None` |
| `["RenameColumn", tableId, oldColId, newColId]` | The final `colId: string` (sanitized/de-duped if needed) |
| `["ModifyColumn", tableId, colId, partialColInfo]` | `None` |
| `["SetDisplayFormula", tableId, fieldRef\|null, colRef\|null, formula]` | `None` — set exactly one of `fieldRef` or `colRef` |
| `["ConvertFromColumn", tableId, srcColId, dstColId, type, widgetOptions, visibleColRef]` | `None` — type-conversion of a column |
| `["CopyFromColumn", tableId, srcColId, dstColId, widgetOptions]` | `None` |
| `["MaybeCopyDisplayFormula", srcColRef, dstColRef]` | `None` |
| `["RenameChoices", tableId, colId, renames]` | `None` — `renames` is `{oldChoice: newChoice}` |
| `["AddEmptyRule", tableId, fieldRef, colRef]` | `None` — adds a blank conditional-style rule |
| `["AddReverseColumn", tableId, colId]` | `{colRef, colId}` for the new RefList back-reference column |

`colInfo` is `{type?: string, isFormula?: bool, formula?: string, label?: string, widgetOptions?: string, visibleCol?: number, rules?, recalcWhen?, recalcDeps?, _position?: number}`.

### Tables

| Action | Returns |
|---|---|
| `["AddEmptyTable", tableId\|null]` | Same shape as `AddTable` (creates 3 default empty/formula columns A,B,C) |
| `["AddTable", tableId, columns[]]` | `{id: number, table_id: string, columns: string[], views: [{id, sections}]}` — `id` is the row in `_grist_Tables`, `table_id` is the final identifier (possibly de-duped), `columns` are the final column ids (excluding the auto-added `manualSort`) |
| `["AddRawTable", tableId\|null]` | Same as `AddEmptyTable` but no primary view/page is created |
| `["RemoveTable", tableId]` | `None` |
| `["RenameTable", oldTableId, newTableId]` | The final `tableId: string` |
| `["DuplicateTable", existingTableId, newTableId, includeData?]` | Same shape as `AddTable` for the new table |

`columns` for `AddTable` is a list of `{id: string\|null, type?: string, isFormula?: bool, formula?: string, label?: string, widgetOptions?: string}`. `id: null` lets the engine pick.

### Views, sections, pages

| Action | Returns |
|---|---|
| `["AddView", tableId, viewType, name]` | `{id: number, sections: number[]}` — `viewType` is `"raw_data"` or `"empty"` |
| `["RemoveView", viewId]` | `None` (deprecated; prefer `RemoveRecord('_grist_Views', viewId)`) |
| `["AddViewSection", title, viewSectionType, viewRowId, tableId]` | `{id: number}` (deprecated; prefer `CreateViewSection`) |
| `["RemoveViewSection", viewSectionId]` | `None` (deprecated; prefer `RemoveRecord`) |
| `["CreateViewSection", tableRef, viewRef, sectionType, groupbyColRefs\|null, tableId]` | `{tableRef: number, viewRef: number, sectionRef: number}` — pass `0` for `tableRef`/`viewRef` to auto-create. `groupbyColRefs` non-null produces a summary section. |
| `["UpdateSummaryViewSection", sectionRef, groupbyColRefs[]]` | `None` — re-group an existing summary section |
| `["DetachSummaryViewSection", sectionRef]` | `None` — convert summary into a regular table |

`sectionType` (a.k.a. `parentKey`) is one of `"record"` (grid), `"detail"` (card list), `"single"` (card), `"chart"`, `"custom"`, `"form"`.

### Imports

| Action | Returns |
|---|---|
| `["GenImporterView", sourceTableId, destTableId, transformRule?, options?]` | A transform-view spec used by the importer UI |

## 5. Worked Examples

### Add a table with three rows in one bundle

```bash
curl -X POST "$HOST/api/docs/$DOC/apply" \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '[
    ["AddTable", "People", [
      {"id": "Name", "type": "Text"},
      {"id": "Age",  "type": "Int"}
    ]],
    ["BulkAddRecord", "People", [null, null, null], {
      "Name": ["Alice", "Bob", "Carol"],
      "Age":  [30, 25, 40]
    }]
  ]'
```

`retValues` will be `[{id:..., table_id:"People", columns:["Name","Age"], ...}, [1,2,3]]`.

### Upsert by primary key

```json
["AddOrUpdateRecord", "People", {"Name": "Alice"}, {"Age": 31}, {"add": true, "update": true}]
```

Returns `{"recordIds":[1], "action":"UPDATE"}` (or `"ADD"` if Alice didn't exist).

### Rename a column

```json
["RenameColumn", "People", "Age", "Years"]
```

Returns `"Years"` (or e.g. `"Years2"` if the name had to be de-duplicated).

## 6. Sharp Edges

- **Negative `rowId`** in an `AddRecord`/`BulkAddRecord` is allowed and lets later actions in the same bundle reference the row before its real id is assigned. The engine remembers the mapping for the duration of the bundle.
- **`?noparse=1`**: by default the server parses string values according to column type (e.g. `"2024-01-01"` → epoch number for Date). Disable when you're sending pre-typed values.
- **Formula columns**: setting values on a formula column raises an error — you must `ModifyColumn` to convert it to data first, or include the column in `require` (not `colValues`) for upserts.
- **Schema actions and ACL**: schema-level UserActions require ACL `schemaEdit` permission; record-level ones require `update` on the affected rows. The check happens inside `_applyUserActions` via `GranularAccess`.
- **System actions** (`Calculate`, `UpdateCurrentTime`, `RespondToRequests`, `RemoveStaleObjects`) bypass normal user-session ACL — they must be applied via `_applyUserActionsAsSystem`, not from the public REST endpoint.
- **`ApplyDocActions` is a power tool**: it skips the higher-level useraction logic (no auto-update of meta-tables, view fields, reverse references, etc.). Use sparingly — almost always prefer the corresponding UserAction.
