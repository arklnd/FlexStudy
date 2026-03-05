# `shared-components` — Architecture Deep Dive

> **Audience**: Engineers building screens or shell UI that need ready-made Hyland-styled Angular components.  
> **Package**: `@hyland/shared-components`  
> **LOC**: 2,888 lines (TS src) · 3,275 spec · 551 HTML · 977 SCSS  
> **Usage score**: 48 import-lines across the repo · Tier 2 — Medium Effort  
> **Companion document**: See [`libs/ARCHITECTURE.md`](../ARCHITECTURE.md) for the full cross-library picture.

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [The Big Picture](#2-the-big-picture)
3. [Component Catalogue](#3-component-catalogue)
   - [Layout Components](#31-layout-components)
   - [Data Display Components](#32-data-display-components)
   - [Navigation Components](#33-navigation-components)
   - [Input Components](#34-input-components)
   - [Overlay and Feedback Components](#35-overlay-and-feedback-components)
   - [Utility Components](#36-utility-components)
4. [Services](#4-services)
5. [Directives and Pipes](#5-directives-and-pipes)
6. [Models and Interfaces](#6-models-and-interfaces)
7. [The HyNgStyleHelper Pattern](#7-the-hyngstylehelper-pattern)
8. [Responsive Layout System](#8-responsive-layout-system)
9. [Data Grid Deep Dive](#9-data-grid-deep-dive)
10. [Design Principles](#10-design-principles)

---

## 1. Purpose

`shared-components` is the **UI building block library** for the OnBase Web Application suite. It provides Angular components that wrap and extend third-party UI primitives (`@hyland/ui`, `@infragistics/igniteui-angular`, Angular Material) with:

- Consistent Hyland branding and theming
- Shadow DOM style isolation for all display components
- Responsive layout behavior driven by a single reactive service
- Grid state persistence (sorting, grouping, selection, column order)
- Integration with `@hyland/flex-router` for navigation-aware components

Components here are consumed by `flex-runtime`, `wv-components`, `wf-approval-mgmt-lib`, and the shell apps.

---

## 2. The Big Picture

```
  Consumer (flex-runtime screen, wv-components, onbase-apps-shell)
  ┌────────────────────────────────────────────────────────────────┐
  │  <hy-ng-layout>, <hy-ng-toolbar>, <hy-ng-data-grid>, ...       │
  └────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
  shared-components Library
  ┌───────────────────────────────────────────────────────────────────┐
  │                                                                   │
  │  Layout              Data Display          Navigation             │
  │  ─────────           ────────────          ──────────             │
  │  HyNgLayout          HyNgDataGrid          HyNgTabs               │
  │  HyNgMasterDetail    HyNgOBADataGrid       ─ FlexRouterLink       │
  │  HyNgSplitter        HyNgText              ─ FlexRouterLinkActive │
  │  HyNgToolbar         HyNgCard                                     │
  │  CollapsibleToolbar  HyNgExpansionPanel    Input                  │
  │  HyNgSpacer          HyNgDivider           ──────────             │
  │                      HyNgTableGroupMenu    HyNgSearchBar          │
  │  Overlay / Feedback  HyNgThemeToggle       HyNgButton             │
  │  ─────────────────                         MenuWithButtons        │
  │  HyNgProgressSpinner                                              │
  │  HyNgInfoDialog                            Utility                │
  │  HyNgErrorMessage                          ──────────             │
  │                                            HyNgStyleHelper        │
  │                                                                   │
  │  HyNgLayoutBehaviorService (responsive isMobileView$)             │
  └────────────────────┬──────────────────────────────────────────────┘
                       │
                       ▼
  Third-party primitives
  ┌────────────────────────────────────────────────────────────────────┐
  │  @hyland/ui · @hyland/ui/material · @infragistics/igniteui-angular │
  │  @angular/material · angular-split · @angular/flex-layout          │
  └────────────────────────────────────────────────────────────────────┘
```

---

## 3. Component Catalogue

### 3.1 Layout Components

| Component | Selector | Encapsulation | Notes |
|-----------|----------|--------------|-------|
| `HyNgLayoutComponent` | *(none — structural)* | ShadowDom | Flex row/column container; inputs: `layout`, `gap`, `useSystemBackground`, `overflowAuto`, `padForScrollBarOnBothSides` |
| `HyNgMasterDetailComponent` | `<hy-ng-master-detail>` | ShadowDom | Two-pane master/detail layout; supports responsive collapse via `HyNgLayoutBehaviorService.isMobileView$`; emits `masterCollapsed` / `masterExpanded`; uses `HyNgSplitterComponent` internally |
| `HyNgSplitterComponent` | `<hy-ng-splitter>` | ShadowDom | Resizable 2-pane panel backed by `angular-split`; inputs: `leftPaneSizeToNotify`, `rightPaneSizeToNotify`; emits `leftPaneReachedSize`, `rightPaneReachedSize` |
| `HyNgToolbarComponent` | `<hy-ng-toolbar>` | ShadowDom | Simple content-projection toolbar wrapper |
| `CollapsibleToolbarComponent` | `<hy-ng-collapsible-toolbar>` | ShadowDom | Toolbar that monitors available width (via `ResizeObserverService`) and collapses overflowing buttons into a `...` overflow menu; input: `items: TemplateRef[]` |
| `HyNgSpacerComponent` | `<hy-ng-spacer>` | ShadowDom | Flex-grow spacer for toolbar alignment |

---

### 3.2 Data Display Components

| Component | Selector | Key Notes |
|-----------|----------|-----------|
| `HyNgDataGridComponent` | `<hy-ng-data-grid>` | Full-featured IgxGrid wrapper — see [§9](#9-data-grid-deep-dive) |
| `HyNgOBADataGridComponent` | `<hy-ng-oba-data-grid>` | OBA-specific variant of HyNgDataGridComponent |
| `HyNgCardComponent` | `<hy-ng-card>` | Styled material card container |
| `HyNgExpansionPanelComponent` | `<hy-ng-expansion-panel>` | IgxExpansionPanel wrapper with Hyland theming |
| `HyNgTableGroupMenuComponent` | `<hy-ng-table-group-menu>` | Dropdown to select a column for grouping in HyNgDataGrid |
| `HyNgTextComponent` | `<hy-ng-text>` | Styled text label with typography variants |
| `HyNgDividerComponent` | `<hy-ng-divider>` | Horizontal or vertical divider |
| `HyNgThemeToggleComponent` | `<hy-ng-theme-toggle>` | Light / dark mode toggle switch |

---

### 3.3 Navigation Components

| Component | Selector | Key Notes |
|-----------|----------|-----------|
| `HyNgTabsComponent` | `<hy-ng-tabs>` | Material tab bar wired to `FlexRouterLink` / `FlexRouterLinkActive` from `@hyland/flex-router`; inputs: `tabs: Tab[]`; emits `tabSelected`; encapsulation: ShadowDom |

---

### 3.4 Input Components

| Component | Selector | Key Notes |
|-----------|----------|-----------|
| `HyNgSearchBarComponent` | `<hy-ng-search-bar>` | Debounced search field backed by `ReactiveFormsModule`; inputs: `debounceTime`, `placeholder`, `appearance`; emits `searchChanged: string`; uses `TranslationPipe` for i18n |
| `HyNgButtonComponent` | `<hy-ng-button>` | Hyland-styled action button |
| `MenuWithButtonsComponent` | `<hy-ng-menu-with-buttons>` | Combined action bar (up to N buttons) + overflow dropdown menu; backed by `CollapsibleToolbarComponent`; inputs: `ButtonConfig[]`, `ButtonGroupConfig[]`; encapsulation: **Emulated** (see §7) |

---

### 3.5 Overlay and Feedback Components

| Component | Selector | Key Notes |
|-----------|----------|-----------|
| `HyNgProgressSpinnerComponent` | `<hy-ng-progress-spinner>` | Centered loading spinner with optional overlay |
| `HyNgInfoDialogComponent` | `<hy-ng-info-dialog>` | Informational modal dialog |
| `HyNgErrorMessageComponent` | `<hy-ng-error-message>` | Inline error banner |

---

### 3.6 Utility Components

| Component | Notes |
|-----------|-------|
| `HyNgStyleHelperComponent` | Adopted by **every** ShadowDom component in this library; injects global Hyland theme CSS (`@hyland/ui` tokens + Angular Material styles) into each shadow root so its children have access to design tokens |

---

## 4. Services

### `HyNgLayoutBehaviorService`

**Token**: `MOBILE_LAYOUT_BEHAVIOR` (`InjectionToken<SupportedBehaviors>`)  
**Provided in**: `root`

Reactive service that drives responsive layout decisions across all components:

```typescript
// Subscribers react to isMobileView$ to alter layout
isMobileView$: Observable<boolean>
currentBehavior$: Observable<SupportedBehaviors>

// Call to switch between Default and Responsive modes
setBehavior(behavior: SupportedBehaviors): void
```

**Configuration**: Inject `MOBILE_LAYOUT_BEHAVIOR` token at module/component level with a `ResponsiveBehavior` value to enable responsive mode:

```typescript
{ provide: MOBILE_LAYOUT_BEHAVIOR, useValue: { type: SupportedTypes.Responsive, mobileQuery: 'lt-md' } }
```

---

### `ResizeObserverService`

Internal service wrapping the native `ResizeObserver` API. Used exclusively by `CollapsibleToolbarComponent` to monitor available width changes.

---

## 5. Directives and Pipes

| Symbol | Type | Purpose |
|--------|------|---------|
| `HyNgMasterSlottedItemDirective` | Directive | Marks content to project into the **master** pane of `HyNgMasterDetailComponent` |
| `HyNgDetailSlottedItemDirective` | Directive | Marks content to project into the **detail** pane of `HyNgMasterDetailComponent` |
| `HyActionBarMenuButtonDirective` | Directive | Internal directive for `MenuWithButtonsComponent` button slotting |

---

## 6. Models and Interfaces

```
  shared-components/src/lib/
  ├── hy-ng-data-grid/models/
  │   ├── DataGridColumnData        column ID, field, header, dataType, width, pinned, filterable…
  │   ├── IDataGridState            serializable snapshot: selectedRowIds, sortingExpressions,
  │   │                             groupedColumnData, columnFieldsOrder, focusedGridCellData
  │   ├── getDefaultGridState()     factory returning a blank IDataGridState
  │   └── IFocusedGridCell          { columnField: string, primaryKeyValue: any }
  └── menu-with-buttons/
      ├── ButtonConfig              { label, action, icon?, isLoading$?, isDisabled$? }
      ├── SingleButtonConfig        single-button variant of ButtonConfig
      ├── ButtonGroupConfig         { label, buttons: ButtonConfig[] } — dropdown group
      └── MenuWithButtonConfiguration  union discriminator for button/group config
```

---

## 7. The HyNgStyleHelper Pattern

All shadow DOM components in this library face a fundamental CSS isolation problem: Web Component shadow roots do not inherit global Angular Material or Hyland design-token styles.

The library solves this by including `HyNgStyleHelperComponent` as a hidden child in every ShadowDom component template:

```
  <hy-ng-style-helper></hy-ng-style-helper>   ← injected at top of every shadow root
  <!-- component content -->
```

`HyNgStyleHelperComponent` queries the document `<head>` for all Hyland theme `<style>` and `<link>` elements and clones them into its own shadow root, effectively "port-forwarding" global styles into each isolated component.

> **Critical exception**: `MenuWithButtonsComponent` uses `ViewEncapsulation.Emulated` instead of ShadowDom because its dropdown menu panel is rendered in a CDK overlay portal **outside** the shadow root and therefore requires global styles to be available normally.

---

## 8. Responsive Layout System

```
  App Bootstrap
      │  optional: provide MOBILE_LAYOUT_BEHAVIOR token
      │  { type: Responsive, mobileQuery: 'lt-md' }
      ▼
  HyNgLayoutBehaviorService
      │  Watches @angular/flex-layout MediaObserver
      │  isMobileView$ = combineLatest([media, behaviorSubject])
      ▼
  HyNgMasterDetailComponent
      │  _isMobileView$ = service.isMobileView$
      │
      ├── isMobile=false: shows two-pane splitter (master | detail)
      └── isMobile=true:  collapses master pane based on leftPaneSizeToNotify threshold
```

If `MOBILE_LAYOUT_BEHAVIOR` is not provided, the service defaults to `SupportedTypes.Default` and `isMobileView$` always emits `false` (static desktop layout).

---

## 9. Data Grid Deep Dive

`HyNgDataGridComponent` wraps `IgxGridComponent` from `@infragistics/igniteui-angular` and adds:

**Features:**

| Feature | Implementation |
|---------|---------------|
| Excel export | `IgxExcelExporterService.export()` |
| Global text filter | Cross-column string/number/date/boolean filter via `FilteringExpressionsTree` |
| Column grouping | `IgxGridComponent.groupBy()` with `HyNgTableGroupMenuComponent` |
| Grid state persistence | `IDataGridState` — serialized and returned via `dataGridStateChange` event |
| Touch support | Converts `touchstart/touchmove/touchend` → `mousedown/mousemove/mouseup` for drag handles |
| Column auto-sizing | `shouldAutoSizeColumns` — fits columns on first paint |
| Multi-select / single-select | `enableMultiselect` input |

**State lifecycle:**

```
  Parent sets [gridState]        Parent sets [data]
        │                               │
        │                        gridVisible = false (destroys grid)
        │                        setTimeout → gridVisible = true
        ▼                               │
  restoreGridState()             ───────┘
        ├── setGroup()
        ├── selectRows()
        ├── rearrangeColumns()
        ├── doSort()
        └── setTimeout → selectCell()
```

**Key outputs:**

| Output | Payload |
|--------|---------|
| `rowDoubleClick` | row data |
| `cellClick` | row data |
| `enterPressedOnCell` | row data |
| `selectedRowsChange` | selected rows array |
| `dataGridStateChange` | `IDataGridState` snapshot |
| `rowCountUpdated` | post-filter row count |
| `filteringDone` / `sortingDone` / `groupingDone` | `void` |
| `deleteKeyUp` | `void` |

---

## 10. Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Shadow DOM by default** | All display components use `ViewEncapsulation.ShadowDom` for style isolation |
| **Style injection bridge** | `HyNgStyleHelperComponent` ports global theme into every shadow root |
| **Single responsive service** | `HyNgLayoutBehaviorService` is the one source of truth for viewport mode |
| **Grid state as data** | `IDataGridState` is serializable — consumers can persist/restore it across page loads |
| **Touch-forward compatibility** | IgxGrid touch polyfills ensure the grid works on tablets |
| **Defer encapsulation when needed** | `MenuWithButtonsComponent` deliberately uses Emulated to allow CDK overlay portal styling |

---

*Generated from codebase analysis — `develop` branch, March 2026.*
