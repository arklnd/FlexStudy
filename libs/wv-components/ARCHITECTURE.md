# `wv-components` — Architecture Deep Dive

> **Audience**: Engineers building or maintaining WorkView object viewing, filter bar, or constraint panel functionality.  
> **Package**: `@hyland/wv-components`  
> **LOC**: 12,844 lines (TS src) · 6,363 spec · 864 HTML · 1,199 SCSS  
> **Usage score**: 36 import-lines across the repo · Tier 1 — Foundational  
> **Companion document**: See [`libs/ARCHITECTURE.md`](../ARCHITECTURE.md) for the full cross-library picture.

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [The Big Picture](#2-the-big-picture)
3. [Library Structure](#3-library-structure)
4. [HyNgWorkViewObjectViewerComponent](#4-hyngworkviewobjectviewercomponent)
5. [Service Architecture](#5-service-architecture)
   - [ViewerShellService](#51-viewershellservice)
   - [WorkViewControllerService](#52-workviewcontrollerservice)
   - [WorkViewControllerObjectService](#53-workviewcontrollerobjectservice)
   - [WorkViewControllerFilterService](#54-workviewcontrollerfilterservice)
   - [WindowMessagingService](#55-windowmessagingservice)
   - [ObjectEventsOrchestratorService](#56-objecteventsorchestratorservice)
   - [UnloadManagerService](#57-unloadmanagerservice)
   - [Supporting Services](#58-supporting-services)
6. [API Service Layer](#6-api-service-layer)
7. [Object Viewer Shell Views and Dialogs](#7-object-viewer-shell-views-and-dialogs)
8. [Constraints System](#8-constraints-system)
9. [Filter Bar System](#9-filter-bar-system)
10. [Form Fields](#10-form-fields)
11. [Layout and Utility Components](#11-layout-and-utility-components)
12. [Cross-Window Messaging Protocol](#12-cross-window-messaging-protocol)
13. [Service Scoping and Lifecycle](#13-service-scoping-and-lifecycle)
14. [Design Principles](#14-design-principles)

---

## 1. Purpose

`wv-components` is the **WorkView object viewing and filtering library** for OnBase Web Applications. It provides:

- **`HyNgWorkViewObjectViewerComponent`** — the main Angular component that renders a WorkView object, acting as an Angular port of the legacy `ObjectViewer.aspx` Web Client page
- **Constraints panel** — a form-based filter criteria editor for WorkView filters
- **Filter bar** — a sidebar navigation list of saved WorkView filters
- **Form field components** — Hyland-styled inputs for WorkView data entry
- **Cross-window messaging** — localStorage-based IPC for coordinating multiple open WorkView objects across browser tabs/windows

---

## 2. The Big Picture

```
  Consumer (Flex screen, wv-components source view)
  ┌──────────────────────────────────────────────────────────────┐
  │  <hy-ng-workview-object-viewer                               │
  │      [classId]="…" [objectId]="…" [viewId]="…">              │
  └──────────────────────────┬───────────────────────────────────┘
                             │  (ObjectEvent output)
                             ▼
  HyNgWorkViewObjectViewerComponent
  ┌──────────────────────────────────────────────────────────────┐
  │  providers: [...objectViewerServices]  ← scoped to component │
  │                                                              │
  │  ViewerShellService        (object session state)            │
  │  WorkViewControllerService (CRUD orchestration)              │
  │  ViewWriterApiService      (HTML rendering API)              │
  │  WorkViewControllerApiService (object/filter API)            │
  │  WindowMessagingService*   (cross-window IPC)                │
  │  ObjectEventsOrchestratorService (events → parent)           │
  │  DialogManagerService      (dialogs within viewer)           │
  └──────────────────────────┬───────────────────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
   ViewWriterApiService          WorkViewControllerApiService
   ┌─────────────────────┐      ┌────────────────────────────────┐
   │ POST /api/workview/ │      │ POST /api/workview/            │
   │  viewwriter/        │      │  workviewcontroller/           │
   │  {method}/          │      │  {method}                      │
   │  httpservicehandler │      └────────────────────────────────┘
   └─────────────────────┘

   * WindowMessagingService is providedIn: root for cross-window coordination
```

---

## 3. Library Structure

```
  wv-components/src/lib/
  ├── constraints/                     Filter constraints UI
  │   ├── hy-ng-constraints-panel/     Main constraints form panel
  │   ├── hy-ng-entry-constraint/      Individual field/operator/value row
  │   └── models/                      ConstraintData, FieldType, AutocompleteData…
  ├── filter-bar/                      Saved filter navigation bar
  │   ├── config/                      Filter bar configuration components
  │   ├── hy-ng-filter-bar-list/       Filter bar list (display)
  │   └── models/                      FilterBar, FilterBarItem, UserSettings…
  ├── form-fields/                     Reusable WorkView form inputs
  ├── hy-ng-column/                    Column layout component
  ├── hy-ng-expander/                  Expandable section
  ├── hy-ng-nav-list/                  Navigation list
  ├── hy-ng-row/                       Row layout component
  ├── hy-ng-workview-object-viewer/    Object viewer (the largest sub-tree)
  │   ├── core/
  │   │   ├── api-services/
  │   │   │   ├── view-writer-api/     ViewWriterApiService (30+ request types)
  │   │   │   └── workview-controller-api/ WorkViewControllerApiService (25+ request types)
  │   │   ├── classes/                 Navigatable
  │   │   ├── enums/                   17 enums (AccessRight, APIRoutes, ControllerContext…)
  │   │   ├── interceptors/            ErrorInterceptor (scoped to AOV)
  │   │   ├── models/                  CommandService, EmbeddedFilter, OpenObject…
  │   │   ├── services/                12 services (see §5)
  │   │   ├── types/
  │   │   └── utilities/               http-utilities, misc-utilities, string-utilities…
  │   ├── views/viewer-shell/
  │   │   └── components/              7 dialog types
  │   ├── hy-ng-workview-object-viewer.component.ts
  │   └── hy-ng-workview-object-viewer-services.ts
  └── pipes/
      └── scoped-async.pipe.ts
```

---

## 4. HyNgWorkViewObjectViewerComponent

**Selector**: `<hy-ng-workview-object-viewer>`  
**Encapsulation**: `ViewEncapsulation.ShadowDom`  
**Change detection**: `OnPush`  
**Implements**: `OnBaseWebServerUnloadableComponentInterface`

The top-level entry point for embedding a WorkView object viewer. It accepts object-identifying inputs and exposes object lifecycle events via `ObjectEventsOrchestratorService.objectEvent$`.

**Key inputs:**

| Input | Type | Description |
|-------|------|-------------|
| `applicationId` | `string` | WorkView Application ID |
| `screenId` | `string` | Flex screen context |
| `classId` | `string` | WorkView Object Class ID |
| `objectId` | `string` | Object primary ID |
| `compositeKey` | `string` | Encoded composite key (when objectId alone insufficient) |
| `viewId` | `string` | View definition ID for rendering |
| `sourceId` | `string` | Source context ID |
| `openedFromContext` | `Context` | How the object was navigated to |
| `createdFromContext` | `Context` | How the object was created |
| `objectSource` | `number` | Object source (coerced from string) |
| `viewerType` | `ViewerType` | `EmbeddedAngular`, `Popup`, etc. |
| `sameWindowNavigation` | `boolean` | Navigate within same tab |
| `isChild` | `boolean` | Is this a child object of another viewer |

---

## 5. Service Architecture

All services except `WindowMessagingService` are declared in `objectViewerServices` and provided **directly on `HyNgWorkViewObjectViewerComponent`** — they are component-scoped and destroyed when the viewer component is destroyed.

### 5.1 ViewerShellService

**The Angular port of `ObjectViewer.aspx`.**

Holds the complete session state for one open WorkView object:
- `classId`, `objectId`, `compositeKey`, `viewId`, `viewerType`, `objectViewerEnvironment`
- `embeddedFilters: EmbeddedFilterWrapper[]`
- `viewerIframe: ElementRef` — reference to the iframe serving the object HTML
- `isViewerReady$: BehaviorSubject<boolean>` — signals when the iframe is ready
- `objectName$: Subject<string>` — current object's display name

Coordinates window messaging, command service connection, and session lifecycle.

---

### 5.2 WorkViewControllerService

**The Angular port of `WorkViewController.js`.**

Main orchestrator for object-level operations:
- `executeRequest$(request)` — delegates to `WorkViewControllerApiService`
- `getNumberOfOpenObjects()` — scans open windows/frames via WindowMessagingService
- Responds to `CheckForAnyOpenWorkViewWindowRequest` messages from other windows

---

### 5.3 WorkViewControllerObjectService

Handles CRUD operations on WorkView objects:

| Operation | Description |
|-----------|-------------|
| `displayObject$` | Load an object by classId+objectId or compositeKey |
| `createObject$` | Create a new object instance |
| `saveObject$` | Save changes to an object |
| `deleteObject$` | Delete an object |
| `copyObject$` | Duplicate an object |
| `resolveObjectId$` | Resolve a composite key to a single objectId |
| `getObjectFormData$` | Retrieve form field metadata for an object |
| `getObjectHistory$` | Retrieve the audit history of an object |
| `getObjectEvents$` | Retrieve workflow/action events |
| `executeServerAction$` | Execute a configured server-side action button |
| `onObjectClosing$` | Signal that the object is about to close |
| `associateObjects$` | Associate related objects |
| `detachRelatedObjects$` | Remove related object association |

---

### 5.4 WorkViewControllerFilterService

Handles WorkView filter operations:
- `executeDynamicObjectQuery$` — run a WorkView filter and get results
- `executeDynamicObjectQueryForCount$` — get result count without data
- `getFilterDataSetViewData$` — get data for embedded filter views
- `saveFilterOverrides$` / `deleteFilterUserOverrides$` — manage user-specific filter overrides
- `getFilterCount$` — count results matching a filter

---

### 5.5 WindowMessagingService

**Provided in root** (unlike all other viewer services).

Enables **cross-window IPC** via `localStorage` events. When a key is written to localStorage in one window, all other windows (that share the same origin) fire a `storage` event.

Pattern:
```
  Window A: publishRequest({ type: CheckForOpenObjectRequest, … })
      → writes WVC_WMS_REQUEST-{uuid} to localStorage

  Window B: storage event fires
      → matches request type
      → publishResponse({ requestId, payload })
      → writes WVC_WMS_RESPONSE-{uuid} to localStorage

  Window A: response Observable filters by matching requestId → resolves
```

This allows one viewer to discover how many objects are open in other separate browser windows.

---

### 5.6 ObjectEventsOrchestratorService

Simple `Subject`-backed event bus that emits `ObjectEvent` values to the viewer component, which then re-emits them to the consumer:

```typescript
enum ObjectEventType {
  DisplayObject = 'display-object',   // open another object
  CloseObject   = 'close-object',     // close the current viewer
}
```

The `DisplayObjectEventData` carries `DisplayObjectProps` (classId, objectId, compositeKey, etc.) so the consumer (e.g. a Flex screen) can open the requested object in a new viewer or the same one.

---

### 5.7 UnloadManagerService

Wraps the Web Client JavaScript `window.UnloadManager` global:
- `registerValidator(fn)` — called before page unload; return `false` to block navigation (e.g. "unsaved changes")
- `registerFinalizer(fn)` — called at unload time for cleanup

---

### 5.8 Supporting Services

| Service | Purpose |
|---------|---------|
| `DialogManagerService` | Creates and tracks Angular Material dialogs inside the viewer (ObjectDialog, TemplateDialog, etc.) |
| `WindowsManagerService` | Manages popup windows; provides `hostWindow` reference for UnloadManager bridge |
| `GlobalObjectStorageService` | Global registry of currently open WV objects (cross-viewer state) |
| `HttpErrorService` | Intercepts 401/403/500 responses for viewer-specific HTTP error handling |
| `LocationHelperService` | URL/location utilities for routing within the viewer context |
| `ObjectServiceBridgeService` | Bridges event-driven messages from the iframe to Angular services |
| `WorkViewControllerLocalizationService` | Loads localized string bundles via `getLocalizedStrings$` |

---

## 6. API Service Layer

Two independent HTTP service facades mirror the WorkView server-side controllers:

### ViewWriterApiService
Base URL: `POST /api/workview/viewwriter/{method}/httpservicehandler`

Executes HTML-rendering requests against the ViewWriter controller. Attaches `X-ViewType` header from `ViewerShellService.viewerType`. Handles both synchronous (XHR) and async (Observable) execution modes.

**Request types** (30+):

| Request | Purpose |
|---------|---------|
| `DisplayObjectRequest` | Render object HTML |
| `GetViewHtmlRequest` | Get view definition HTML |
| `GetTemplateContentRequest` | Fetch template for dialogs |
| `ExecuteServerActionRequest` | Trigger a server action |
| `GetDataSetValuesRequest` | Fetch dataset autocomplete values |
| `GetFilterDataSetViewDataRequest` | Get embedded filter view data |
| `GetRelatedObjectViewDataRequest` | Load related objects tab |
| `LiveUpdateRequest` | Polling for object live updates |
| `GetSubscriptionStatusRequest` | Check notification subscription |
| `FilterCreateExcelDocumentRequest` | Export filter results to Excel |
| `FilterCreatePrintPageRequest` | Create print-ready view |

---

### WorkViewControllerApiService
Base URL: `POST /api/workview/workviewcontroller/{method}`

Handles data operations:

| Request | Purpose |
|---------|---------|
| `CreateObjectRequest` | Create a new WV object |
| `SaveObjectRequest` | Persist object changes |
| `DeleteMultipleObjectsRequest` | Bulk delete |
| `CopyObjectRequest` | Duplicate object |
| `ResolveObjectIdRequest` | Resolve composite key |
| `GetObjectFormDataRequest` | Get form metadata |
| `GetObjectActionsRequest` | Get available actions |
| `GetObjectHistoryRequest` | Fetch audit log |
| `ExecuteDynamicObjectQueryRequest` | Run a filter |
| `GetDependentObjectInfoRequest` | Get related object info |
| `CheckClassRightsRequest` | Verify user permissions |
| `UpdateUrlRequest` | Update browser URL for current object |

---

## 7. Object Viewer Shell Views and Dialogs

The `ViewerShellView` is the inner rendered view inside `HyNgWorkViewObjectViewerComponent`. It manages dialogs created during object viewer interactions:

| Dialog | Trigger |
|--------|---------|
| `ObjectDialogComponent` | Opening an object in a modal |
| `DocumentsDialogComponent` | Displaying associated documents |
| `ObjectHistoryDialogComponent` | Object audit history (grid view) |
| `TemplateDialogComponent` | General-purpose message box (like Windows MessageBox) |
| `WorkflowDialogComponent` | Workflow action dialogs |

**`TemplateDialogComponent`** supports:
- `MessageBoxButtonEnum`: OK, OKCancel, YesNo, YesNoCancel, RetryCancel, AbortRetryIgnore
- `MessageBoxImageEnum`: None, Question, Information, Warning, Error, Check
- `MessageBoxResultEnum`: None, OK, Cancel, Yes, No, Retry, Abort, Ignore

**`ObjectHistoryDialogComponent`** has its own internal data grid component (`GridComponent`, `GridColumnComponent`, `GridContainerComponent`) for rendering history rows.

---

## 8. Constraints System

The constraints system provides a form-based filter criteria editor for WorkView filters:

```
  HyNgConstraintsPanelComponent
  ├── HyNgEntryConstraintComponent  (one per constraint row)
  │     ├── Field selector
  │     ├── OperatorMenuComponent   (equals, contains, startsWith, between…)
  │     └── Value input             (type-appropriate: text, date, autocomplete…)
  └── AND / OR toggle (MatButtonToggle)
```

**Models:**

| Type | Description |
|------|-------------|
| `ConstraintData` | One constraint row: `{ field, operator, value, secondValue }` |
| `FieldType` | Enum: Text, Numeric, Date, DateTime, Boolean, Alphanumeric… |
| `FieldVisualizationType` | How the field should be rendered (TextBox, DatePicker, Dropdown…) |
| `ConstraintDropDownType` | Dropdown data source type |
| `AutocompleteData` | `{ items: T[], displayField, valueField }` for autocomplete sources |
| `ConstraintPromptForFilterQuery` | Prompt definition for filter query constraints |
| `DataSourceType` | Enum for constraint dropdown data source |

---

## 9. Filter Bar System

The filter bar provides a sidebar navigation list of saved WorkView filters:

```
  HyNgFilterBarListComponent           — displays filter bars + items
  HyNgFilterBarConfigListComponent     — configure filter bar visibility / ordering

  Data models:
  FilterBar          { id, name, items: FilterBarItem[] }
  FilterBarItem      { id, name, filterCount$, isActive }
  ApplicationFiltersUserSettings  { hidden: id[], order: id[] }
```

**Key outputs:**

| Output | Payload |
|--------|---------|
| `filterBarItemSelected` | `FilterBarItem` |
| `filterBarsLoaded` | `boolean` |
| `countRequested` | `GetFilterBarCountRequest` (callers must provide count via `GetFilterBarCountResponse`) |

The filter bar uses a **pull-the-count** pattern: it emits `countRequested` and the consumer calls back with `GetFilterBarCountResponse` rather than issuing HTTP calls itself, keeping the network logic in the consuming component.

---

## 10. Form Fields

Hyland-styled form fields for WorkView data entry:

| Component | Selector | Input type |
|-----------|----------|-----------|
| `HyNgFormFieldComponent` | `<hy-ng-form-field>` | Base/text field |
| `HyNgAutocompleteFormFieldComponent` | `<hy-ng-autocomplete-form-field>` | Typeahead autocomplete |
| `HyNgDateFormFieldComponent` | `<hy-ng-date-form-field>` | Date picker |
| `HyNgDatetimeFormFieldComponent` | `<hy-ng-datetime-form-field>` | Date + time picker |
| `HyNgSelectFormFieldComponent` | `<hy-ng-select-form-field>` | Static dropdown |

All form fields integrate with Angular Reactive Forms (`AbstractControl`) and `FormFieldErrorService` from `@hyland/flex-forms`.

---

## 11. Layout and Utility Components

| Symbol | Type | Description |
|--------|------|-------------|
| `HyNgRowComponent` | Component | Flex-row container for WorkView layouts |
| `HyNgColumnComponent` | Component | Flex-column container for WorkView layouts |
| `HyNgExpanderComponent` | Component | Collapsible/expandable section panel |
| `HyNgNavListComponent` | Component | Styled navigation list (filter bar items) |
| `ScopedAsyncPipe` | Pipe | `async`-like pipe that is scoped to a component's change detection |
| `decodeAndParseKey(key)` | Utility | Decodes base64/JSON encoded composite keys from WorkView Client |
| `Navigatable` | Class | Abstract base class for navigatable WV objects; exposes `navigate(direction, props)` |

---

## 12. Cross-Window Messaging Protocol

`WindowMessagingService` solves the problem of coordinating between multiple WorkView objects that may be open in different browser windows/tabs (not iframes):

```
  localStorage key format:
  WVC_WMS_REQUEST-{uuid}   → request (set by window that initiates)
  WVC_WMS_RESPONSE-{uuid}  → response (set by other windows listening)

  Other windows receive 'storage' event, handle via handleStorageEvent()
  The initiating window also fires events via requestSubject / responseSubject

  WindowMessageTypes:
  ├── CheckForOpenObjectRequest    — ask others if they have an object open
  ├── CheckForAnyOpenWorkViewWindowRequest — ask if any WV window is open
  ├── CloseWindowRequest           — request closure of another window
  └── … (extensible via ResponseMapping type)
```

Because `storage` events only fire in **other** windows (not the setting window), the service also directly feeds requests/responses into its own subjects when it sets the values.

---

## 13. Service Scoping and Lifecycle

This is a critical architectural decision:

```typescript
// HyNgWorkViewObjectViewerComponent metadata:
providers: objectViewerServices   // ← ALL object viewer services listed here

// WHY: Services hold per-object state (classId, objectId, viewerIframe, etc.)
// This state MUST NOT survive when the viewer is closed and re-opened for a different object.
// Providing at component level ensures Angular destroys and recreates services with each viewer instance.
```

**Exception**: `WindowMessagingService` is `providedIn: 'root'` because it listens to `window.storage` events and must be shared across all viewer instances in the same browser window.

---

## 14. Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Port fidelity** | `ViewerShellService` and `WorkViewControllerService` are direct TypeScript ports of `ObjectViewer.aspx` and `WorkViewController.js` — preserving battle-tested Web Client logic |
| **Component-scoped services** | All viewer services are scoped to the viewer component so state is always fresh per-object |
| **Dual API abstraction** | `ViewWriterApiService` (HTML rendering) and `WorkViewControllerApiService` (data operations) mirror the server's two-controller pattern |
| **Cross-window IPC via localStorage** | `WindowMessagingService` uses localStorage as a cross-origin-same-origin event bus since `postMessage` is unreliable across unrelated windows |
| **Event-driven object lifecycle** | `ObjectEvent` (DisplayObject / CloseObject) flows through `ObjectEventsOrchestratorService` so the parent screen controls navigation without tight coupling |
| **Pull-the-count pattern** | Filter bar emits count requests to consumers rather than issuing HTTP calls — keeps network logic in one place |
| **Shadow DOM isolation** | `HyNgWorkViewObjectViewerComponent`, `HyNgConstraintsPanelComponent`, `HyNgFilterBarListComponent` all use `ViewEncapsulation.ShadowDom` |

---

*Generated from codebase analysis — `develop` branch, March 2026.*
