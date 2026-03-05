# `onbase-web-server` — Architecture Deep Dive

> **Audience**: Engineers working on OnBase content, documents, workflow, or building new source views.  
> **Package**: `@hyland/onbase-web-server`  
> **LOC**: 6,874 lines (TS src) · 919 spec · 604 HTML · 525 SCSS  
> **Usage score**: 31 import-lines across the repo · Tier 1 — Foundational  
> **Companion document**: See [`libs/ARCHITECTURE.md`](../ARCHITECTURE.md) for the full cross-library picture.

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [The Big Picture](#2-the-big-picture)
3. [Source View Architecture](#3-source-view-architecture)
4. [Module Structure](#4-module-structure)
5. [Content Modules](#5-content-modules)
   - [Content Viewer](#51-content-viewer)
   - [Document Querying](#52-document-querying)
   - [Search Panel](#53-search-panel)
   - [Keyword Panel](#54-keyword-panel)
   - [Folder Viewer](#55-folder-viewer)
   - [Workflow Item Viewer](#56-workflow-item-viewer)
   - [File Upload](#57-file-upload)
   - [Reporting Dashboards](#58-reporting-dashboards)
   - [Unity Forms](#59-unity-forms)
6. [Source Views (NgRx Slices)](#6-source-views-ngrx-slices)
7. [Shared Infrastructure](#7-shared-infrastructure)
   - [Dialog Service](#71-dialog-service)
   - [License Consumption Service](#72-license-consumption-service)
   - [BlockingService](#73-blockingservice)
   - [WebClientThemeService](#74-webclientthemeservice)
8. [HTTP API Pattern](#8-http-api-pattern)
9. [Design Principles](#9-design-principles)

---

## 1. Purpose

`onbase-web-server` is the **primary application layer library** that bridges Angular components with the OnBase Web Server API. It provides:

- **Source view components** for each type of OnBase content (documents, folders, workflow queues, WorkView objects)
- **Content viewers** that host Web Client iframe panels
- **Workflow and query services** for interacting with OnBase backends
- **NgRx store slices** for each source view, managing search results, loading state, and selection
- **Shared infrastructure**: dialog management, license acquisition, navigation blocking, and theme integration

This library is the integration layer between the Flex runtime screen system (flex-runtime, flex-config) and the actual OnBase content APIs.

---

## 2. The Big Picture

```
  Flex Runtime (screen renders source views)
  ┌───────────────────────────────────────────────────────────────┐
  │  <document-query-search-sv>  <workflow-queue-sv>  <error-sv>  │
  │  <folder-query-search-sv>    <workview-filter-sv>  …          │
  └────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
  onbase-web-server Source Views (NgRx + Components)
  ┌───────────────────────────────────────────────────────────────┐
  │  DocumentQuery   FolderQuery   Workflow   WorkView            │
  │  WorkView-SingleObj   Error  (each: store + effects + module) │
  └─────────────────────────────┬─────────────────────────────────┘
                                │ NgRx Actions / Effects
                                ▼
  onbase-web-server Services (HTTP)
  ┌────────────────────────────────────────────────────────────────┐
  │  ContentQueryService  WorkflowQueryService  SourceQueryService │
  │  WorkflowTaskService  LicenseConsumptionService                │
  └─────────────────────────────┬──────────────────────────────────┘
                                │ HTTP
                                ▼
  OnBase Web Server API (/api/…)
```

---

## 3. Source View Architecture

A **source view** is a route-activated component that represents a specific OnBase content mode. All source views follow the same structural pattern:

```
  source-views/<name>/
  ├── components/
  │   └── <name>.sv.component.ts        — Angular component, selector: <name>-sv
  ├── state-store/
  │   ├── <name>-store.models.ts         — State type definition (Immutable<{…}>)
  │   ├── <name>.store.ts                — createFeature (actions + reducer + selectors)
  │   └── <name>.effects.ts             — NgRx Effects: Action → HTTP → Action
  └── <name>.module.ts                  — NgModule (lazy-loadable)
```

**Uniform state shape** (varies by view but always includes):

```typescript
type <Name>State = Immutable<{
  queryResults:  DataSource<ResultType>;
  isLoading:     LoadingStatus;          // Idle | Loading | Loaded | Error
  errorMessage:  string | undefined;
  viewState:     IViewStateBase | undefined;
}>;
```

**NgRx Effect data-flow**:

```
  User triggers search
      │
      ▼
  Store.actions.executeSearch dispatched
      │
      ▼
  <Name>Effects.executeQuery$
      │  debounceTime(10)
      │  switchMap → HttpQueryService.execute$()
      │  takeUntil(Store.actions.selectDocument)   ← cancels on selection change
      ▼
  Store.actions.updateQueryResults / handleQueryError dispatched
      │
      ▼
  Reducer updates immutable state (via immer produce())
      │
      ▼
  Component selects state → renders results
```

---

## 4. Module Structure

```
  OnBaseWebServerModule
  ├── OnBaseWebServerServicesModule   — services only (for environments without UI)
  ├── WorkflowModule                  — workflow item viewer + services
  ├── SourceQueryModule               — source query service + licensing
  └── (lazy-loaded source view modules registered by consumers)

  Source view NgModules (each standalone/lazy-loadable):
  ├── DocQueryModule
  ├── FolderQueryModule
  ├── WorkflowSourceModule
  ├── WorkViewSourceModule
  └── WorkViewSingleObjectSourceModule
```

---

## 5. Content Modules

### 5.1 Content Viewer

**Component**: `ViewerComponent` (`<hy-ng-viewer>`)  
**Base class**: `OnBaseWebServerControlWrapperComponent`

Embeds a Web Client iframe to render OnBase document content. The component:
- Uses `WebServerSourceDirective` to set the `src` attribute safely
- Injects `WEB_CLIENT_WINDOW`, `WEB_CLIENT_APP_HOST_CONFIG` (from `@hyland/onbase-web-server-core`) to communicate with the Web Client JavaScript bridge
- Tracks `docId` changes via `OnChanges` → sends message to Web Client frame to navigate to new document
- Exposes `documentName$: Observable<string>` and `itemUpdated$: Subject<void>` for parent consumption

---

### 5.2 Document Querying

**Service**: `ContentQueryService`  
**API endpoints**: `/api/CustomQueryType/:id`, `/api/QueryResults`, `/api/FolderQueryResults`

Constructs XML-based query tokens and executes document or folder searches:

```typescript
constructQuery(
  queryItemName: QueryItemName,
  queryCategories: QueryCategory[],
  keywords: string,
  startDate?: Date, endDate?: Date,
  fullTextSearchInput?: string,
): string   // returns XML query token

executeDocumentBasedQuery$(queryType, criteria): Observable<DataSource<ContentQueryDocResult>>
executeFolderBasedQuery$(criteria): Observable<DataSource<ContentQueryFolderResult>>
```

**Result types**:
- `ContentQueryDocResult` — document result row (ID, name, doc type, keywords…)
- `ContentQueryFolderResult` — folder result row
- `DataSource<T>` — union of `QueryResults<T>` and `CustomQueryFolderResults<T>`

**Component**: `ContentDataGridComponent` — `HyNgDataGridComponent` specialized for content query results with a custom `ContentDataGridFilteringStrategy`.

---

### 5.3 Search Panel

**Component**: `SearchPanelComponent`

Hosts the keyword panel and date-range inputs. Emits `SearchExecutedEvent` when the user submits a query. Consumed by source view components.

**Sub-components:**
- `DatePickerPanelComponent` — single-date filter
- `DateRangePanelComponent` — date range filter

---

### 5.4 Keyword Panel

**Component**: `KeywordPanelComponent`  
**Service**: `KwpRenderingService`

Renders OnBase keyword metadata fields. Supports three modes via `KeywordPanelMode`:
- `Retrieval` — for standard document type queries
- `CustomQuery` — for saved custom query execution
- `FolderQuery` — for folder content queries

**Config types**: `KeywordPanelConfig` → `RetrievalConfig | CustomQueryConfig | FolderCustomQueryConfig`

---

### 5.5 Folder Viewer

**Component**: `FolderViewerComponent`  
Shows the content of an OnBase folder. Uses `ContentQueryService` for folder-level query results.

---

### 5.6 Workflow Item Viewer

**Component**: `WorkItemViewerComponent` — renders a work item inside an iframe (like `ViewerComponent`)  
**Component**: `WorkflowItemViewerComponent` — manages the workflow item display context  
**Service**: `WorkflowTaskService` — performs workflow task operations (save, route, etc.)  
**Service**: `WorkflowQueryService` — retrieves queue results and count

**Work item menu components:**
- `WorkItemTasksMenuComponent` — toolbar for current work items
- `WorkItemInQueueTasksMenuComponent` — queued-item task actions
- `RelatedWorkItemTasksMenuComponent` — related-document task actions

**Workflow licensing** (see §7.2): each queue interaction acquires/releases a workflow license via `LicenseConsumptionService`.

---

### 5.7 File Upload

**Component**: `FileUploadEnhancedComponent`  
Enhanced drag-and-drop and file-browser upload component for OnBase document ingestion.

---

### 5.8 Reporting Dashboards

**Component**: `DashboardViewerComponent`  
Renders an OnBase reporting dashboard embedded view.

**Events emitted by dashboard interactions:**
- `DocumentSelectedEventData` — user selected a document from dashboard
- `FolderSelectedEventData` — user selected a folder from dashboard

---

### 5.9 Unity Forms

**Component**: `CreateUnityFormComponent`  
Launches an OnBase Unity Form for document creation or workflow initiation.

---

## 6. Source Views (NgRx Slices)

| Source View | Store Name | Component | Key Effect |
|------------|-----------|-----------|-----------|
| `doc-query` | `DocumentQuery` | `DocumentQuerySearchSourceViewComponent` | `executeDocumentQuery$` → `ContentQueryService` |
| `folder-query` | `FolderQuery` | `FolderQuerySearchSourceViewComponent` | `executeFolderQuery$` → `ContentQueryService` |
| `workflow` | `WorkflowQueue` | `WorkflowQueueSourceViewComponent` | `executeQueueQuery$` → `WorkflowQueryService` |
| `workview` | `WorkView` | `WorkViewFilterSourceViewComponent`, `WorkViewSearchSourceViewComponent` | `executeWorkViewQuery$` → WorkView API |
| `workview-single-object` | `WorkViewSingleObject` | `WorkViewSingleObjectSourceViewComponent` | `loadSingleObject$` → WorkView API |
| `error` | *(no store)* | `ErrorSourceViewComponent` | *(stateless error display)* |

Each store follows the **same 5-action shape**:
1. `executeSearch` — initiate a query
2. `selectDocument` / `selectItem` — select a result row
3. `updateQueryResults` — store results
4. `updateViewState` — persist selection/scroll/filter state between navigations
5. `handleQueryError` — store an error message

---

## 7. Shared Infrastructure

### 7.1 Dialog Service

**Service**: `OnBaseWebServerDialogService`  
**Module**: `OnBaseWebServerDialogModule`

Opens Angular Material dialogs hosting OnBase components:

```typescript
// Full-screen (100vw × 100vh) — e.g. document viewer in popup
openFullscreenDialog<T>(config: FullScreenDialogData<T, K>): MatDialogRef

// Fit-content modal — e.g. confirmation or keyword entry dialog
openModalDialog<T>(config: ModalScreenDialogData<T>): MatDialogRef
```

Both use `closeOnNavigation: false` to prevent the dialog auto-closing during Web Client iframe navigations.

Container components: `FullScreenDialogContainerComponent`, `ModalDialogContainerComponent`

---

### 7.2 License Consumption Service

**Service**: `LicenseConsumptionService`  
**Token**: `LICENSE_CONSUMPTION_HANDLER_PROVIDER` (multi-provider array of `LicenseConsumptionHandler`)

Serializes license acquire/release requests via a `BehaviorSubject` pipeline:

```
  Source View opens (e.g. WorkflowQueue)
       │
       ▼
  LicenseConsumptionService.acquireLicense(LicensableModule.Workflow, sourceId)
       │  debounceTime(25) → prevents transient acquire+release on rapid navigations
       │  concatMap → handler.acquireLicense$(licModule, sourceId)
       ▼
  acquireLicenseSubject$.next(response)

  Source View closes
       │
       ▼
  releaseLicense(LicensableModule.Workflow, sourceId)
       │  concatMap → handler.releaseLicense$(…)
       ▼
  releaseLicenseSubject$.next(response)
```

**Licensable modules**: `LicensableModule.Generic`, `LicensableModule.Workflow`

---

### 7.3 BlockingService

**Service**: `BlockingService` implements `CanActivate`

A registry of `CanActivate` guards that can collectively block Angular Router navigation. Consumers register/deregister themselves:

```typescript
blockingService.register(myGuard);    // add to blocking set
blockingService.deregister(myGuard);  // remove from blocking set
// Angular calls blockingService.canActivate() → polls all registered guards
```

Used by source views to block navigation while unsaved work is in progress.

---

### 7.4 WebClientThemeService

**Service**: `WebClientThemeService`

Reads the active Web Client theme selection and applies corresponding Angular Material theme CSS class to the document body. Ensures the Angular UI matches the Web Client's chosen color scheme.

---

## 8. HTTP API Pattern

All query services use `HttpClient` directly:

| Service | Endpoint | Method |
|---------|---------|--------|
| `ContentQueryService` | `/api/QueryResults` | POST (XML body) |
| `ContentQueryService` | `/api/FolderQueryResults` | POST (XML body) |
| `ContentQueryService` | `/api/CustomQueryType/:id` | GET |
| `WorkflowQueryService` | `/api/obapps/queue` | POST |
| `WorkflowQueryService` | `/api/obapps/queue/count` | POST |
| `SourceQueryService` | `/api/source-query/:id` | GET |

Error responses are wrapped in `ProblemDetailError` (extends native `Error`) from `ProblemDetail` models.

HTTP requests that require authentication use `WebServerRequestInterceptor` which attaches the Web Client session context headers.

---

## 9. Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Source view isolation** | Each source view is a self-contained NgModule with its own NgRx store slice |
| **Uniform store shape** | All store slices share the same 5-action pattern and `Immutable<{…}>` state type |
| **Immer immutability** | `produce()` from `immer` wraps all reducer logic; state mutations are written imperatively |
| **Cancellable effects** | `takeUntil(cancelAction$)` in effects prevents stale results from old HTTP requests |
| **License lifecycle** | `LicenseConsumptionService` serializes acquire/release via `concatMap` + `debounceTime` |
| **Iframe hosting** | `OnBaseWebServerControlWrapperComponent` base class + `WebServerSourceDirective` safely bridge Angular → Web Client iframe |
| **Dialog safety** | `closeOnNavigation: false` prevents Angular router events from closing embedded viewer dialogs |

---

*Generated from codebase analysis — `develop` branch, March 2026.*
