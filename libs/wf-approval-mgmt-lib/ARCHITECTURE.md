# `wf-approval-mgmt-lib` — Architecture Deep Dive

> **Audience**: Engineers working on the Workflow Approval Management (WAM) application or integrating approval process configuration.  
> **Package**: `@hyland/wf-approval-mgmt-lib`  
> **LOC**: 6,341 lines (TS src) · 7,281 spec · 547 HTML · 345 SCSS  
> **Usage score**: 23 import-lines across the repo · Tier 2 — Feature Library  
> **Companion document**: See [`libs/ARCHITECTURE.md`](../ARCHITECTURE.md) for the full cross-library picture.

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [The Big Picture](#2-the-big-picture)
3. [Domain Model](#3-domain-model)
   - [Approval Process Hierarchy](#31-approval-process-hierarchy)
   - [Rule and Condition System](#32-rule-and-condition-system)
4. [NgRx State Architecture](#4-ngrx-state-architecture)
   - [Store Slices](#41-store-slices)
   - [Facade Pattern](#42-facade-pattern)
5. [HTTP Services](#5-http-services)
6. [Components](#6-components)
   - [Alert Components](#61-alert-components)
   - [Conditions Tree Editor](#62-conditions-tree-editor)
   - [Supporting Components](#63-supporting-components)
7. [Supporting Services](#7-supporting-services)
8. [i18n and Constants](#8-i18n-and-constants)
9. [Design Principles](#9-design-principles)

---

## 1. Purpose

`wf-approval-mgmt-lib` is the **feature library** for the Workflow Approval Management (WAM) product. It provides a complete Angular + NgRx implementation of the UI and data layer for:

- **Listing** all approval processes configured in OnBase
- **Viewing and editing** the detail of an approval process: its paths, levels, approvers, and routing rules
- **Configuring conditions** on rules using a visual drag-and-drop tree editor
- **Managing approvers**: replacing, importing via CSV, and assigning by user or approval role

The library is consumed by `workflow-approval-mgmt` — the standalone Flex application entry point for WAM.

---

## 2. The Big Picture

```
  workflow-approval-mgmt app (screens)
  ┌───────────────────────────────────────────────────────────────┐
  │  Approval Process List screen                                 │
  │  Approval Process Detail screen                               │
  └─────────────────────────────┬─────────────────────────────────┘
                                │ inject facades
                                ▼
  wf-approval-mgmt-lib Facades
  ┌────────────────────────────────────────────────────────────────┐
  │  ApprovalProcessesFacadeService    (list CRUD)                 │
  │  ApprovalProcessDetailsFacadeService  (detail + conditions)    │
  └──────────────┬ dispatch / select ──────────────────────────────┘
                 │
                 ▼
  NgRx Store (feature: 'approvalProcesses')
  ┌───────────────────────────────────────────────────────────────────┐
  │  approvalProcesses       Status<ApprovalProcessesModel>           │
  │  approvalProcessDetails  StatusObject<ApprovalProcessDetailModel> │
  │  approvalUserRoleList    Status<ApproverModel>                    │
  └──────────────┬ Effects ───────────────────────────────────────────┘
                 │
                 ▼
  ApprovalProcessesService (HTTP)
  ┌────────────────────────────────────────────────────────────────┐
  │  GET  /api/wam/processes                                       │
  │  GET  /api/wam/processes/:id                                   │
  │  POST /api/wam/processes/:id/paths/sequence                    │
  │  POST /api/wam/processes/:id/paths/:pathId/levels/sequence     │
  │  POST /api/wam/processes/:id/paths                             │
  │  … (full CRUD for paths, levels, rules, approvers)             │
  └────────────────────────────────────────────────────────────────┘
```

---

## 3. Domain Model

### 3.1 Approval Process Hierarchy

```
  ApprovalProcessDetailModel
  ├── approvalProcessId / Name
  ├── autoApprovalPaths: AutoApprovalPathModel[]   — automatic routing
  ├── approvalPaths: ApprovalPathModel[]            — manual approval routing
  │     └── ApprovalPathModel
  │           ├── approvalPathId / approvalRuleId / seqNumber
  │           └── approvalLevels: ApprovalLevelModel[]
  │                 └── ApprovalLevelModel
  │                       ├── approvalLevelId / approvalRuleId / seqNumber
  │                       ├── approvalType: ApprovalLevelTypeEnum
  │                       │     AllMustApprove=1 | AnyoneCanApprove=2
  │                       ├── approvers: ApproverModel[]
  │                       │     └── { approverType, approverId, approverName }
  │                       │         ApproverTypeEnum: User=1 | ApprovalRole=3
  │                       └── approverUserRules: ApprovalUserRuleModel[]
  ├── pathRules: ApprovalRuleModel[]               — rules for path selection
  ├── levelRules: ApprovalRuleModel[]              — rules for level selection
  ├── userRules: ApprovalRuleModel[]               — rules for user assignment
  ├── availableDocumentTypes: DocumentTypeModel[]
  ├── availableKeywordTypes: KeywordModel[]        — with KeywordDataTypeEnum
  ├── availableFormFields: UnityFormFieldModel[]
  └── availableUnityScripts: UnityScriptModel[]
```

**`ApprovalPathTypeEnum`**: `AllUserMustApprove=1 | AnyUserCanApprove=2`

---

### 3.2 Rule and Condition System

Rules gate path selection, level resolution, and user assignment. Each rule carries a condition tree:

```
  ApprovalRuleModel
  ├── approvalRuleId / Name / description
  ├── approvalType: ApprovalRuleTypeEnum
  │     Path=1 | Level=2 | UserRule=3
  └── conditions: ConditionModel[]  (polymorphic)
        ├── GroupConditionModel        — AND/ALL or OR/ANY logical group
        ├── KeywordConditionModel      — keyword field comparison (==, !=, <, >, contains…)
        ├── DocumentTypeConditionModel — document type match
        ├── UnityFormConditionModel    — Unity Form field comparison
        └── UnityScriptConditionModel  — run a Unity Script for custom logic
```

**`ConditionTypeEnum`**:
```
  GroupConditions=1, EvaluateDocumentType=2, EvaluateKeywordValue=3,
  EvaluateUnityFormFieldValue=4, RunUnityScript=5
```

**`OperatorEnum`**: standard comparison operators (equals, notEquals, lessThan, greaterThan, contains, startsWith, endsWith)

**`KeywordDataTypeEnum`**: Null, LargeNumeric, AlphaNumeric, Currency, Date, Float, SmallNumeric, DateTime, and case-sensitivity variants

The condition tree is rendered and edited using the visual `HyNgConditionsComponent` (see §6.2).

---

## 4. NgRx State Architecture

### 4.1 Store Slices

**Feature name**: `'approvalProcesses'` (constant `APPROVAL_PROCESSES`)

```typescript
interface State {
  approvalProcesses:      Status<ApprovalProcessesModel>;
  approvalProcessDetails: StatusObject<ApprovalProcessDetailModel>;
  approvalUserRoleList:   Status<ApproverModel>;
}

// Status<T>  = { error: string|null; result: T[] }
// StatusObject<T> = { error: string|null; result: T }
```

**Three reducer slices, each following the same Request/Success/Failure pattern:**

| Slice | Actions |
|-------|---------|
| `approvalProcesses` | `loadApprovalProcessesRequest/Success/Failure` |
| `approvalProcessDetails` | `loadApprovalProcessDetailsRequest/Success/Failure` + CRUD actions |
| `approvalUserRoleList` | `loadApprovalUserRoleRequest/Success/Failure` |

**Selectors** (all via `createFeatureSelector('approvalProcesses')`):
- `getApprovalProcesses` → `Status<ApprovalProcessesModel>`
- `getApprovalProcessDetail` → `StatusObject<ApprovalProcessDetailModel>`
- `getApprovalUsersOrRoles` → `Status<ApproverModel>`

**NgRx Effects** call `ApprovalProcessesService` HTTP methods, show success/failure snackbars via `SnackbarService`, and dispatch result actions.

---

### 4.2 Facade Pattern

Components do not interact with the NgRx store directly. Two facades serve as the public API:

**`ApprovalProcessesFacadeService`** (list operations):
```typescript
getApprovalProcesses(): Observable<Status<ApprovalProcessesModel>>
loadApprovalProcesses(): void     // dispatches loadApprovalProcessesRequest
loadingApprovalProcess$: BehaviorSubject<boolean>
```

**`ApprovalProcessDetailsFacadeService`** (detail + in-memory editing operations):
```typescript
// Store-backed selectors
getApprovalProcessDetail(): Observable<StatusObject<ApprovalProcessDetailModel>>
getApprovalUsersOrRoles(): Observable<Status<ApproverModel>>

// In-memory state for conditions editing
ConditionDataChange: BehaviorSubject<ConditionsNode[]>
selectedNode: BehaviorSubject<any>

// Tree CRUD operations
addPathToProcessDetails(…): void
addLevelToPathDetails(…): void
addApproverToLevel(…): void
removeApproverFromLevel(…): void
replaceApprover(…): void
addRuleToPath(…): void

// Form helpers
createApproverFormGroup(): FormGroup
createPathFormGroup(): FormGroup
```

`ApprovalProcessDetailsFacadeService` also holds significant in-memory mutable state (tree editing, condition `id` counter, `treeData`) which is not stored in NgRx. This is editing scratch-pad state that is dispatched to the store on save.

---

## 5. HTTP Services

**`ApprovalProcessesService`** — all calls go to `/api/wam/…`:

| Operation | HTTP | Endpoint |
|-----------|------|---------|
| List processes | GET | `/api/wam/processes` |
| Get process detail | GET | `/api/wam/processes/:id` |
| Update path sequence | POST | `/api/wam/processes/:id/paths/sequence` |
| Update level sequence | POST | `/api/wam/processes/:id/paths/:pathId/levels/sequence` |
| Add approval path | POST | `/api/wam/processes/:id/paths` |
| Add level to path | POST | `/api/wam/processes/:id/paths/:pathId/levels` |
| Add approver to level | POST | `…/levels/:lvlId/approvers` |
| Remove approver | DELETE | `…/approvers/:approverId` |
| Replace approver | PUT | `…/approvers/:approverId` |
| Save rule conditions | POST/PUT | `…/rules/:ruleId/conditions` |
| Import CSV approvers | POST | `/api/wam/processes/:id/import` |

**`SourceQueryService`** (from `@hyland/onbase-web-server`) is also referenced for resolving OnBase user data when searching for approvers.

---

## 6. Components

### 6.1 Alert Components

| Component | Selector | Purpose |
|-----------|----------|---------|
| `HyNgModalComponent` | `<hy-ng-modal>` | Generic WAM dialog opened via Angular Material `MatDialog`; input data: `{ header, message, confirmButtonText, dismissButtonText, customContent, saveFunction, parameters }` |
| `HyNgSimpleAlertComponent` | `<hy-ng-simple-alert>` | Inline info/success/warning banner |
| `HyNgRemoveAlertComponent` | `<hy-ng-remove-alert>` | Confirmation banner: "Are you sure you want to remove…?" |

---

### 6.2 Conditions Tree Editor

**Component**: `HyNgConditionsComponent` (`<hy-ng-conditions>`)

Renders an editable **nested tree** of approval rule conditions using Angular Material CDK Tree + DragDrop:

```
  Tree structure (ConditionsNode[])
  ├── Group (ALL)
  │     ├── EvaluateKeywordValue: Invoice Amount > 10000
  │     └── Group (ANY)
  │           ├── EvaluateDocumentType: Invoice
  │           └── RunUnityScript: ValidateVendor
  └── EvaluateUnityFormFieldValue: Department = "Finance"
```

**Features:**
- Drag-and-drop reordering (`CdkDragDrop`)
- Add/remove condition nodes
- Inline form editing for keyword values (with `KeywordDataTypeEnum`-aware input types)
- Add/remove group nodes (AND/ALL or OR/ANY logical groupings)
- Reads `availableDocumentTypes`, `availableKeywordTypes`, `availableFormFields`, `availableUnityScripts` from facade

State is managed through `ApprovalProcessDetailsFacadeService.ConditionDataChange$`.

---

### 6.3 Supporting Components

| Component | Selector | Purpose |
|-----------|----------|---------|
| `HyNgReplaceApproverComponent` | `<hy-ng-replace-approver>` | Dialog for searching and selecting a replacement approver (User or ApprovalRole) |
| `HyNgImportCsvComponent` | `<hy-ng-import-csv>` | CSV file upload for bulk approver import |
| `HyNgFilepickerComponent` | *(internal)* | File selection input for CSV upload |

---

## 7. Supporting Services

| Service | Purpose |
|---------|---------|
| `SnackbarService` | Wraps Angular Material `MatSnackBar`; exposes `showSuccessMessage()` and `showFailureMessage()` |
| `TranslateService` | Maps string keys from the i18n bundle to display strings |
| `ProcessDataService` | Data transformation helpers (normalizing API responses, building tree structures from flat lists) |

---

## 8. i18n and Constants

**i18n**: Translation strings are loaded statically from `./assets/i18n/en-us` — no external translation loading. `translateStrings.wamtc` contains all UI text used in effects, components, and dialogs.

**Constants** (`utils.ts`): String constants for NgRx action type names (`LOAD_APPROVAL_PROCESSES_REQUEST`, etc.), regular expression patterns (`PATTERN`), and UI constants (`CONSTANT_VALUE`).

---

## 9. Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Facade isolation** | Components inject only facades, never the NgRx store directly |
| **Request/Success/Failure triad** | Every async operation has exactly three actions — eliminates ad-hoc loading state management |
| **Status<T> wrapper** | All store state uses `Status<T>` / `StatusObject<T>` to co-locate data and error together |
| **In-memory editing scratch pad** | Condition tree edits live in facade memory until explicitly saved — no partial saves trigger store updates |
| **Hierarchical condition model** | Conditions form a recursive tree (groups containing groups) to support arbitrarily complex approval routing logic |
| **Snackbar-first feedback** | All async effects show immediate snackbar on success or failure before dispatching the result action |

---

*Generated from codebase analysis — `develop` branch, March 2026.*
