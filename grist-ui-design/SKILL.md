---
name: grist-ui-design
description: Use when building or styling UI components in the Grist codebase — cards, lists, icons, badges, accordions, typography, spacing, hover effects, or any styled component. Also use when unsure which design tokens, CSS patterns, or component conventions to follow in Grist.
---

# Grist UI Design Reference

## Overview

Grist uses **grainjs** `styled()` for all components. Design tokens come from `theme.*` (colors) and `vars.*` (typography/spacing). Never hardcode colors — always use tokens. Modifier classes use the `cls("-modifier")` pattern.

```typescript
import { theme, vars } from "app/client/ui2018/cssVars";
import { icon } from "app/client/ui2018/icons";
import { styled } from "grainjs";
```

---

## Color Tokens (`theme.*`)

| Token | Use |
|---|---|
| `theme.text` | Primary text |
| `theme.lightText` | Secondary/muted text, hints, timestamps |
| `theme.inputBorder` | Borders, dividers, card outlines |
| `theme.pageBg` | Page/section background (off-white) |
| `theme.mainPanelBg` | Left/side panel background |
| `theme.controlFg` | Links, anchor text (green) |
| `theme.controlPrimaryBg` | Primary green — icons, active states |
| `theme.controlSecondaryFg` | Hover border for cards |
| `theme.selectionOpaqueBg` | Light green tint — icon bg, selected state |
| `theme.errorText` | Error red |

**Exception — hardcoded tints** (only for status/semantic use):
```typescript
background: rgba(208, 2, 27, 0.08);  // error bg tint
background: #FFF3E0; color: #E65100;  // warning (unsaved)
background: #e8f5e9; color: #2e7d32;  // success badge
```

---

## Typography (`vars.*`)

| Token | Use |
|---|---|
| `vars.xsmallFontSize` | Monospace payloads, pill badges |
| `vars.smallFontSize` | Labels, hints, sub-text |
| `vars.mediumFontSize` | Default body text |
| `vars.xxxlargeFontSize` | Page headings/breadcrumbs |
| `vars.headerControlTextWeight` | Heading font weight |

**Weight rules:**
- `font-weight: 500` — medium emphasis (card titles, labels)
- `font-weight: 600` — strong emphasis (section headings, breadcrumb current)
- Accordion headers: `text-transform: uppercase; letter-spacing: 0.06em;`

---

## Spacing

Standard values: **4, 6, 8, 10, 12, 16, 20, 24px**

- Icon size: **28×28px** (inline), **36×36px** (detail headers)
- Card padding: **12px** (bordered cards), **14px 16px** (flat list rows)
- Section padding: **20px**
- Gap between icon and text: **10px**
- Gap between buttons: **8px**

---

## Icons

```typescript
icon("Mail")        // email action
icon("Code")        // webhook action
icon("Dropdown")    // accordion arrow, expand chevron
icon("Filter")      // filter
icon("Warning")     // alerts
icon("Remove")      // delete
icon("FieldColumn") // column picker
```

Set color via CSS variable — **never use `color:` on icons**:
```typescript
const cssIconWrap = styled("div", `
  --icon-color: ${theme.controlPrimaryBg};
`);
```

**Icon background container (standard):**
```typescript
const cssActionIcon = styled("div", `
  width: 28px;
  height: 28px;
  border-radius: 6px;
  background: ${theme.selectionOpaqueBg};
  display: flex;
  align-items: center;
  justify-content: center;
  --icon-color: ${theme.controlPrimaryBg};
  flex-shrink: 0;
`);
```

---

## Component Patterns

### Bordered Card (interactive, e.g. Actions)
```typescript
const cssCard = styled("div", `
  border: 1px solid ${theme.inputBorder};
  border-radius: 8px;
  padding: 12px;
  margin-bottom: 8px;
  cursor: pointer;
  &:hover {
    border-color: ${theme.controlSecondaryFg};
    box-shadow: 0 2px 8px rgba(0,0,0,0.04);
  }
`);
```

### Flat List Row (Event Log / dense lists)
```typescript
const cssRow = styled("div", `
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 14px 16px;
  border-bottom: 1px solid ${theme.inputBorder};
  cursor: pointer;
  &:hover {
    border-color: ${theme.controlPrimaryBg};  // green divider on hover
  }
`);
```

> **Hover rule:** Bordered cards → box-shadow + border-color. Flat rows → change border-color (green). Never use background fill as hover for rows.

### Expandable Row (inline detail)
```typescript
// Card gets -expanded modifier, detail panel follows immediately
cssRow.cls("-expanded"),  // removes bottom border
// detail panel:
const cssDetail = styled("div", `
  padding: 12px 16px;
  border-bottom: 1px solid ${theme.inputBorder};
  background: ${theme.pageBg};
`);
```

### Accordion Header
```typescript
const cssAccordionHeader = styled("div", `
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 16px;
  font-size: ${vars.mediumFontSize};
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: ${theme.lightText};
  background: ${theme.pageBg};
  border-top: 1px solid ${theme.inputBorder};
  border-bottom: 1px solid ${theme.inputBorder};
  margin-top: -1px;
  cursor: pointer;
  &:hover { color: ${theme.text}; }
`);
// Arrow: rotate(-90deg) closed, rotate(0deg) open
```

### Badge / Pill
```typescript
const cssBadge = styled("span", `
  font-size: ${vars.xsmallFontSize};
  font-weight: 500;
  padding: 2px 8px;
  border-radius: 999px;
  white-space: nowrap;
  &-enabled  { background: ${theme.selectionOpaqueBg}; color: ${theme.controlPrimaryBg}; }
  &-disabled { background: ${theme.pageBg}; color: ${theme.lightText}; }
  &-error    { background: rgba(208,2,27,0.08); color: ${theme.errorText}; }
`);
```

### Dashed Add Button
```typescript
const cssAddBtn = styled("div", `
  border: 1.5px dashed ${theme.inputBorder};
  border-radius: 8px;
  padding: 10px;
  text-align: center;
  color: ${theme.lightText};
  cursor: pointer;
  font-size: ${vars.smallFontSize};
  &:hover {
    border-color: ${theme.controlPrimaryBg};
    color: ${theme.controlPrimaryBg};
    background: ${theme.selectionOpaqueBg};
  }
`);
```

### Radio / Option Cards
```typescript
const cssOption = styled("label", `
  display: flex;
  align-items: flex-start;
  gap: 10px;
  padding: 10px 12px;
  border: 1.5px solid ${theme.inputBorder};
  border-radius: 8px;
  cursor: pointer;
  margin-bottom: 6px;
  &-selected {
    border-color: ${theme.controlPrimaryBg};
    background: ${theme.selectionOpaqueBg};
  }
`);
// Radio accent: style="accent-color: #16b378"
```

---

## Two-Line Card Row Layout

Standard layout for action/log rows (icon + title/sub + right meta):
```typescript
dom("div", { style: "display: flex; align-items: center; gap: 10px;" },
  cssIcon(icon("Mail")),
  dom("div", { style: "flex: 1; min-width: 0;" },
    dom("div", { style: "font-weight: 500; font-size: 12px;" }, title),
    dom("div", { style: `font-size: ${vars.smallFontSize}; color: ${theme.lightText};` }, sub),
  ),
  cssRightSlot(/* badge, time, arrow */),
)
```

Text overflow for sub-line:
```typescript
overflow: hidden;
text-overflow: ellipsis;
white-space: nowrap;
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Hardcoding `#16b378` for green | Use `theme.controlPrimaryBg` |
| `color:` on icon elements | Use `--icon-color:` CSS variable |
| Background fill on hover for flat rows | Use `border-color` change instead |
| `border-radius: 4px` for cards | Cards use `8px`; only small elements use `4px` |
| Forgetting `flex-shrink: 0` on icons | Always add to prevent icon squish |
| Using `margin-top` spacing in lists | Use `margin-bottom` on each item or `gap` on parent |
