# `flex-shared` — Architecture Deep Dive

> **Audience**: Engineers working anywhere in the Flex library stack.  
> **Package**: `@hyland/flex-shared`  
> **LOC**: 2,041 lines (TS src) · 504 spec · 1 HTML · 0 SCSS  
> **Usage score**: 195 import-lines across the repo · Tier 1 — CANNOT REMOVE  
> **Companion document**: See [`libs/ARCHITECTURE.md`](../ARCHITECTURE.md) for the full cross-library picture.

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [The Big Picture](#2-the-big-picture)
3. [Architecture: Six Zones](#3-architecture-six-zones)
4. [Zone 1 — DI Tokens + Service Interfaces](#4-zone-1--di-tokens--service-interfaces)
   - [Core Runtime Services](#41-core-runtime-services)
   - [Communication Services](#42-communication-services)
   - [Routing Services](#43-routing-services)
   - [Tracking Services](#44-tracking-services)
5. [Zone 2 — Domain Interfaces & Models](#5-zone-2--domain-interfaces--models)
6. [Zone 3 — Concrete Service Implementations](#6-zone-3--concrete-service-implementations)
7. [Zone 4 — Enums](#7-zone-4--enums)
8. [Zone 5 — Utilities](#8-zone-5--utilities)
   - [stringFormat](#81-stringformat)
   - [TaskQueue / QueueItem](#82-taskqueue--queueitem)
   - [noopAction](#83-noopaction)
9. [Zone 6 — Mocks & Test Utilities](#9-zone-6--mocks--test-utilities)
10. [DI Token Map](#10-di-token-map)
11. [Design Principles](#11-design-principles)

---

## 1. Purpose

`flex-shared` is the **dependency-inversion hub** of the Flex stack. Almost every other library either:
- **Publishes** its service behind a token here (`flex-router` provides `FLEX_ROUTER_SERVICE`)
- **Consumes** a service declared here (`flex-operations` injects `FLEX_LOG_SERVICE`)

Without `flex-shared`, every cross-library service reference would be either a direct class import (creating tight coupling and circular dependencies) or duplicated.

Its secondary role is to house **small, critical utilities** (`stringFormat`, `TaskQueue`) and **all mock implementations** of every service interface, so test setup across the entire codebase requires only one import source.

---

## 2. The Big Picture

```
  The Flex dependency graph (simplified):
  
  flex-config  flex-types
       │             │
       └──────┬──────┘
              │
        flex-shared  ←  Every library in the stack imports from here
              │
       ┌──────┼─────────────────────┬─────────────┐
       ▼      ▼                     ▼             ▼
  flex-forms  flex-operations  flex-router   flex-runtime
                                                  │
                                             onbase-web-server
                                                  │
                                           onbase-apps-host, client-app
  
  flex-shared owns:
  ○ 20+ DI tokens (one per cross-library service interface)
  ○ All service interfaces as TypeScript contracts
  ○ Concrete implementations of HTTP, logging, expression evaluation
  ○ Mock implementations of ALL service interfaces (for tests)
  ○ Small utilities: stringFormat, TaskQueue, noopAction
```

---

## 3. Architecture: Six Zones

```
  flex-shared/src/lib/
  ├── interfaces/              Zone 1: DI tokens + service interface declarations
  │   ├── services/            (all service interfaces + tokens)
  │   ├── context.ts
  │   ├── control-http-error.ts
  │   ├── flex-screen-snapshot.model.ts
  │   ├── operation-cancellation-token.ts
  │   ├── operation-execution-args.ts
  │   ├── operation-type.ts
  │   ├── property-change.interface.ts
  │   └── route-options.ts
  ├── services/                Zone 3: Concrete implementations
  │   ├── expression-evaluator/
  │   ├── http-service/
  │   ├── log-service/
  │   └── operation-state-store/
  ├── enums/                   Zone 4: Domain enums
  ├── components/              Zone 5: FlexStyleHelperComponent
  ├── mocks/                   Zone 6: All mock implementations
  ├── test-utils/              Zone 6: Test helpers + testing modules
  ├── utils/                   Zone 5: stringFormat, TaskQueue, noopAction
  └── flex-constants.ts        Zone 2: Application-wide constants
```

---

## 4. Zone 1 — DI Tokens + Service Interfaces

### 4.1 Core Runtime Services

| Token | Interface | Purpose |
|-------|-----------|---------|
| `FLEX_LOG_SERVICE` | `ILogService` | Structured logging (debug, info, warn, error) |
| `FLEX_CONFIG_SERVICE` | `IConfigurationService` | Access to `FlexRuntimeConfiguration` |
| `FLEX_RUNTIME_CONFIG_SERVICE` | `IFlexRuntimeConfigService` | Set/get the runtime config; `setConfig(config)` |
| `FLEX_AUTHENTICATION_SERVICE` | `IAuthService` | Auth token retrieval |
| `FLEX_HTTP_SERVICE` | `IHttpService` | HTTP client abstraction (GET/POST/PUT/DELETE) |
| `FLEX_DIALOG_SERVICE` | `IDialogService` | Open/close modal dialogs |
| `FLEX_TOAST_SERVICE` | `IToastService` | Show toast notifications |
| `FLEX_PROPERTY_BINDING_SERVICE` | `IPropertyBindingService` | Resolve runtime property binding values |
| `FLEX_CONTROL_INIT_SERVICE` | `IFlexControlInitService` | Initialize flex controls on screen load |
| `EXPRESSION_EVALUATOR_SERVICE` | `IExpressionEvaluatorService` | Evaluate string expressions against context |
| `EXPRESSION_ASSIGNMENT_OVERRIDES` | `IExpressionAssignmentOverrides` | Override expression evaluation results |
| `ENVIRONMENT_VARIABLES_SERVICE` | `IEnvironmentVariablesService` | Access `envData` from the runtime config |
| `OPERATION_EXECUTION_SERVICE` | `IOperationExecutionService` | Execute an operation list by ID |
| `OPERATION_EXECUTOR_SERVICE` | `IOperationExecutorService` | Execute a single operation step |

---

### 4.2 Communication Services

| Token | Interface | Purpose |
|-------|-----------|---------|
| `FLEX_BROADCAST_SERVICE` | `IBroadcastService` | Cross-frame pub/sub message bus |
| `FLEX_TOAST_SERVICE` | `IToastService` | Show success/error/info toast messages |

**`BroadcastMessage`** — the typed message envelope:

```typescript
interface BroadcastMessage {
  type: BroadcastType;
  payload?: any;
}
```

**`BroadcastType`** — enum of message types understood by the Flex host.

---

### 4.3 Routing Services

| Token | Interface | Purpose |
|-------|-----------|---------|
| `FLEX_ROUTER_PARAMETER_SERVICE` | `IRouterParameterService` | Access current route parameters |

**`RouteParamDictionary`** — typed `{ [paramName: string]: string }` map.

**`RouterParameterChanged`** — emitted when a route param changes: `{ name: string; oldValue: string; newValue: string }`.

**`RouteDataType`** / **`RouteQueryParamHandling`** — enums for route configuration options.

---

### 4.4 Tracking Services

| Token | Interface | Purpose |
|-------|-----------|---------|
| `COMPONENT_TRACKING_SERVICE` | `IComponentTrackingService` | Track which component instances are on the current screen |

**`TrackedComponent`** — `{ id: string; element: HTMLElement; instance: any }`.

---

## 5. Zone 2 — Domain Interfaces & Models

| Type | Description |
|------|-------------|
| `IContext` | The execution context passed to expression evaluator: `{ $store, $global, $event }` |
| `IOperationType` | Contract for a flex operation type class: `{ operationType: string; execute(args): Promise<any> }` |
| `IOperationExecutionArgs` | Arguments passed to `IOperationType.execute()`: config data, context, cancellation token |
| `OperationCancellationToken` | Passed into operations; if cancelled, `throw new OperationCancellationError()` |
| `OperationCancellationError` | Custom error class thrown to signal cancellation (not a real failure) |
| `PropertyChange` | `{ property: string; oldValue: any; newValue: any }` |
| `ControlHttpError` | Holds HTTP error status + response body for display in error state |
| `FlexScreenSnapshot` | Snapshot of a screen's state at a point in time |
| `FlexConstants` | App-wide constants (`FlexConstants.FLEX_CONTROL_ID_ATTR`, etc.) |

---

## 6. Zone 3 — Concrete Service Implementations

Three concrete services are provided (not just interfaces):

### `FlexLogService`
Angular-injectable logger that wraps `console.*` with:
- Minimum log level filtering (from `FlexRuntimeConfiguration.minimumLoggingLevel`)
- Structured prefix format: `[category] message`

```typescript
this.logService.debug('[operation]', 'Executing HTTP request');
this.logService.error('[operation]', 'HTTP failed', error);
```

### `HttpService`
Wraps Angular `HttpClient` with the `FLEX_HTTP_SERVICE` interface contract. Passes through headers, handles error mapping to `ControlHttpError`.

### `OperationStateStore`
In-memory key/value store that provides local and global scopes for operation execution:
- **Local scope**: values live only for the duration of a single operation list run
- **Global scope**: values persist for the lifetime of the current screen instance

Read via `$store.<key>` and `$global.<key>` in expression strings.

### `ExpressionAssignmentOverrides`
Mechanism for test or integration code to pre-override expression evaluation results without needing real backend data.

---

## 7. Zone 4 — Enums

| Enum | Values | Usage |
|------|--------|-------|
| `Connector` | `And \| Or` | Logical connectors in condition expressions |
| `ConstraintPromptOperator` | `equals`, `startsWith`, `contains`, etc. | Filter operators for keyword searches |
| `ShortcutKeycodes` | Key code constants | Keyboard shortcut definitions |

---

## 8. Zone 5 — Utilities

### 8.1 `stringFormat`

Named-parameter string interpolation (like C#'s `string.Format`):

```typescript
import { stringFormat } from '@hyland/flex-shared';

stringFormat('Hello {0}, you have {1} items', ['World', 3]);
// → 'Hello World, you have 3 items'
```

Used by `TranslationPipe` in `onbase-web-server-core` for LRM string interpolation.

---

### 8.2 `TaskQueue` / `QueueItem`

A concurrency-limiting async task queue:

```typescript
const queue = new TaskQueue(1);  // max 1 concurrent task

const item = queue.queueFunctionAndRunIfAbleTo(async () => {
  return await fetchData();
});

await item.promise;  // resolves when this item's task completes

queue.clearQueue();  // cancels all pending (not started) items
```

**Used in**: `flex-runtime`'s operation list runner to serialize operation list executions when `shouldCancelPreviousInvocationsOfList = true`.

`QueueItem` has three lifecycle states:
1. **Queued** — promise pending, not yet started
2. **Running** — `.startItem()` called, callback executing
3. **Finished/Cancelled** — promise resolved/rejected

---

### 8.3 `noopAction`

A zero-argument `() => void` function that does nothing. Used as a placeholder in NgRx action definitions where a real reducer side effect is not needed.

```typescript
const clearErrors = createAction('[Flex] Clear Errors');
// implementation just uses noopAction as the default handler
```

---

## 9. Zone 6 — Mocks & Test Utilities

Every service interface has a corresponding mock class in `src/lib/mocks/`:

| Mock | Implements | Behavior |
|------|-----------|---------|
| `MockAuthService` | `IAuthService` | Returns empty token |
| `MockBroadcastService` | `IBroadcastService` | Records published messages |
| `MockComponentTrackingService` | `IComponentTrackingService` | In-memory component registry |
| `MockConfigurationService` | `IConfigurationService` | Returns injected config |
| `MockControlInitService` | `IFlexControlInitService` | Records init calls |
| `MockDialogService` | `IDialogService` | Returns configurable dialog result |
| `MockEnvironmentVariablesService` | `IEnvironmentVariablesService` | Returns static env vars |
| `MockFlexHttpService` | `IHttpService` | Returns pre-configured responses |
| `MockFlexRuntimeConfigService` | `IFlexRuntimeConfigService` | Stores config in memory |
| `MockLogService` | `ILogService` | Silent or spy-able logger |
| `MockOperationExecutionService` | `IOperationExecutionService` | Records executed list IDs |
| `MockOperationExecutorService` | `IOperationExecutorService` | Records executed operations |
| `MockOperationStateStore` | (internal) | In-memory store for tests |
| `MockPropertyBindingService` | `IPropertyBindingService` | Returns static bindings |
| `MockToastService` | `IToastService` | Records toasts |
| `MockExpressionEvaluator` | `IExpressionEvaluatorService` | Returns literal expressions |
| `MockExpressionOverride` | `IExpressionAssignmentOverrides` | Records overrides |
| `MockRouterParameter` | `IRouterParameterService` | Returns static route params |

**Test utilities** in `src/lib/test-utils/`:

| Utility | Description |
|---------|-------------|
| `FlexSharedTestingModule` | NgModule that provides all mocks — one import for a complete test bed |
| `JasmineHelpers` | Typed `spyOn` helpers and assertion utilities |
| `TestingUtilities` | DOM query helpers, fixture management |
| `TranslocoTestingModule` | Pre-configured translocoTestingModule for i18n test beds |

---

## 10. DI Token Map

```
  Producing library      Token                           Consuming libraries
  ─────────────────────  ──────────────────────────────  ───────────────────────────────────
  flex-runtime           FLEX_CONFIG_SERVICE             flex-operations, flex-forms, flex-router
  flex-runtime           FLEX_RUNTIME_CONFIG_SERVICE     flex-host, onbase-web-server
  onbase-web-server      FLEX_AUTHENTICATION_SERVICE     flex-operations, wv-components
  flex-runtime           FLEX_HTTP_SERVICE               flex-operations, wv-components
  flex-runtime           FLEX_DIALOG_SERVICE             flex-operations, onbase-web-server
  flex-runtime           FLEX_TOAST_SERVICE              flex-operations
  flex-runtime           FLEX_LOG_SERVICE                flex-forms, flex-operations, flex-router
  flex-runtime           FLEX_BROADCAST_SERVICE          flex-operations, onbase-web-server
  flex-runtime           FLEX_PROPERTY_BINDING_SERVICE   flex-operations
  flex-runtime           EXPRESSION_EVALUATOR_SERVICE    flex-operations
  flex-runtime           OPERATION_EXECUTION_SERVICE     flex-operations, flex-router
  flex-operations        OPERATION_EXECUTOR_SERVICE      flex-runtime
  flex-router            FLEX_ROUTER_SERVICE             flex-runtime, flex-operations
  flex-router            FLEX_ROUTER_PARAMETER_SERVICE   flex-operations, onbase-web-server
  flex-runtime           COMPONENT_TRACKING_SERVICE      flex-operations
```

---

## 11. Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Dependency Inversion** | All cross-library service refs go through tokens; no direct class imports |
| **Zero circular deps** | `flex-shared` depends only on `flex-config` and `flex-types` |
| **Test parity** | One mock per interface; `FlexSharedTestingModule` for zero-friction test setup |
| **Concurrency control** | `TaskQueue` prevents concurrent operation list execution races |
| **Cancellation support** | `OperationCancellationToken` + `OperationCancellationError` pattern |
| **Consistent logging** | All Flex libraries use `FLEX_LOG_SERVICE` for unified, level-filtered output |

---

*Generated from codebase analysis — `develop` branch, March 2026.*
