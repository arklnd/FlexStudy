# `flex-operations` — Architecture Deep Dive

> **Audience**: Engineers implementing new operation types or debugging operation execution pipelines.  
> **Package**: `@hyland/flex-operations`  
> **LOC**: 2,736 lines (TS src) · 3,388 spec · 22 HTML · 14 SCSS  
> **Usage score**: 26 import-lines across the repo · Tier 3 — Moderate  
> **Companion document**: See [`libs/ARCHITECTURE.md`](../ARCHITECTURE.md) for the full cross-library picture.

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [The Big Picture](#2-the-big-picture)
3. [Architecture Summary](#3-architecture-summary)
4. [Execution Pipeline](#4-execution-pipeline)
5. [Operation Type Registry](#5-operation-type-registry)
6. [OperationExecutorService](#6-operationexecutorservice)
7. [OperationExecutionService](#7-operationexecutionservice)
8. [Operation State Store (NgRx)](#8-operation-state-store-ngrx)
9. [OperationStateStoreProxy](#9-operationstatestoredproxy)
10. [All 18 Operation Types](#10-all-18-operation-types)
11. [Adding a New Operation Type](#11-adding-a-new-operation-type)
12. [Cancellation System](#12-cancellation-system)
13. [Design Principles](#13-design-principles)

---

## 1. Purpose

`flex-operations` is the **operation execution engine** of the Flex runtime. It implements the 18 built-in operation types that configuration authors can chain together into operation lists. 

An operation list is a named sequence of steps (like a macro) that runs in response to events: component `@Output()` events, lifecycle hooks, or form submissions. This library provides:
1. An **extensible registry** of operation type classes
2. The **execution engine** that walks operation lists step by step
3. An **NgRx feature state slice** for the operation state store
4. All **18 built-in operation types**

---

## 2. The Big Picture

```
  flex-config
  ┌─────────────────────────────────┐
  │  OperationListConfiguration[]   │  (declarative step lists)
  │  OperationEventMapping[]        │  (event → list wiring)
  └──────────────────┬──────────────┘
                     │
                     ▼
  flex-runtime dispatches: "component emitted event X, run operation list Y"
                     │
                     ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │  flex-operations                                                  │
  │                                                                   │
  │  OperationExecutionService (OPERATION_EXECUTION_SERVICE token)    │
  │  └── resolves list ID → OperationExecutorService.execute()        │
  │                                                                   │
  │  OperationExecutorService (OPERATION_EXECUTOR_SERVICE token)      │
  │  ├── for each OperationConfiguration in list:                     │
  │  │     ├── check cancellation token                               │
  │  │     ├── registry.getOperationInstance(type)                    │
  │  │     └── instance.execute(context, config, args)                │
  │  └── catches errors → log + toast, breaks list on error           │
  │                                                                   │
  │  OperationTypeRegistryService                                     │
  │  └── string operationType → IOperationType instance               │
  │                                                                   │
  │  OperationStore (NgRx)                                            │
  │  └── Dictionary<any>  — in-screen key/value state for operations  │
  └───────────────────────────────────────────────────────────────────┘
```

---

## 3. Architecture Summary

```
  flex-operations/src/lib/
  ├── operations/              18 built-in operation type implementations
  │   ├── bind-form-data/
  │   ├── broadcast-message-to-flex-host/
  │   ├── compare/
  │   ├── console-log/
  │   ├── display-alert/
  │   ├── display-component-dialog/
  │   ├── display-dialog/
  │   ├── display-toast/
  │   ├── execute-http-request/
  │   ├── execute-script/
  │   ├── invoke-method/
  │   ├── invoke-method-on-control/
  │   ├── navigate-to-route/
  │   ├── set-control-visibility/
  │   ├── set-flex-control-input/
  │   ├── set-property/
  │   ├── set-property-from-control-property/
  │   └── set-property-from-route-data-param/
  ├── services/
  │   ├── operation-execution.service/     ← OPERATION_EXECUTION_SERVICE token impl
  │   ├── operation-executor.service/      ← OPERATION_EXECUTOR_SERVICE token impl
  │   ├── operation-type-registry.service/ ← maps string type → class instance
  │   ├── expression-evaluator/            ← JSON expression evaluator
  │   └── operation-execution-utilities.ts
  ├── state-store/
  │   ├── operation-store.ts              ← NgRx feature slice (OperationStore)
  │   └── operation-state-store.proxy.ts  ← NgRx Effects bridge to flex-shared
  ├── models/
  │   └── rest-api-operations/http-response.ts
  ├── flex-operations.module.ts
  └── test-utils/
```

---

## 4. Execution Pipeline

```
  1. User interaction (component @Output() or lifecycle hook fires)
              │
              ▼
  2. flex-runtime maps event → operationListId via OperationEventMapping
              │
              ▼
  3. OperationExecutionService.executeOperationList(listId, context, args)
              │
              ▼
  4. OperationExecutorService.execute(listId, context, args)
     ├── Looks up OperationListConfiguration from FlexRuntimeConfiguration
     ├── If shouldCancelPreviousInvocationsOfList:
     │     └── Cancel any existing CancellationToken for this listId
     └── For each OperationConfiguration in the list:
           ├── Check cancellationToken.isCancelled → throw OperationCancellationError
           ├── registry.getOperationInstance(op.type) → IOperationType
           ├── operation.execute(context, op.config, args) [async]
           └── On error:
                 ├── If OperationCancellationError → rethrow (propagate up)
                 └── Else: log error + show toast, break the list
              │
              ▼
  5. Each IOperationType.execute() may:
     ├── Read/write the OperationStateStore ($store, $global)
     ├── Call other services via injected tokens
     └── Return a value stored via storeAs config
```

---

## 5. Operation Type Registry

`OperationTypeRegistryService` is the **factory registry** that maps operation type strings to their implementation class instances:

```typescript
class OperationTypeRegistryService {
  registerOperationType(name: string, instance: IOperationType): void
  getOperationInstance(name: string): IOperationType
}
```

Each operation type registers itself when `FlexOperationsModule` is imported:

```typescript
@NgModule({
  providers: [
    // All 18 operation types are registered in providers via factory pattern
  ]
})
export class FlexOperationsModule {}
```

External libraries can register custom operations types by providing them to the registry.

---

## 6. OperationExecutorService

Implements `IOperationExecutorService` (from `flex-shared`). Provided via `OPERATION_EXECUTOR_SERVICE` token.

Key design choices:
- **Sequential execution**: operations in a list run `await`-sequentially, not in parallel
- **Break-on-error**: if any operation throws (except cancellation), the list stops
- **Cancellation propagation**: `OperationCancellationError` bubbles up through compare operations
- **Per-list tokens**: cancellation is tracked by `operationListId` so only one token per list is active

```typescript
tryExecute(operationListId, context, args): Promise<boolean>
// Returns true = success, false = error occurred (never throws)

execute(operationListId, context, args): Promise<void>
// Throws OperationCancellationError if cancelled; logs and toasts on other errors
```

---

## 7. OperationExecutionService

The higher-level service that connects events to operation lists. Implements `IOperationExecutionService`.

Typically used by `flex-runtime` to run an operation list by ID when an event mapping fires.

---

## 8. Operation State Store (NgRx)

Operations read and write values through the `OperationStateStore` (from `flex-shared`). The underlying NgRx slice is defined in `OperationStore`:

```
  NgRx Store
  ┌──────────────────────────────────────────────────────────┐
  │  'Operations' feature slice                              │
  │  State: Dictionary<any>   (string key → any value)       │
  │                                                          │
  │  Actions:                                                │
  │  ├── [Operations] Set/Update Value { name, value }       │
  │  │     → Immer produce: state[name] = value              │
  │  └── [Operations] Clear Store                            │
  │        → returns {}                                      │
  └──────────────────────────────────────────────────────────┘
```

Uses **Immer** (`produce()`) for immutable state updates.

Operations access the store using expression syntax:
- `$store.propertyName` — local scope (per operation list run)
- `$global.propertyName` — global scope (per screen instance)

---

## 9. OperationStateStoreProxy

An NgRx **Effect** that bridges between `OperationStore.actions` and `flex-shared`'s `OperationStateStore` service. Ensures that consumers injecting `FLEX_PROPERTY_BINDING_SERVICE` can observe state changes without directly depending on NgRx.

---

## 10. All 18 Operation Types

| Operation Type | Class | Key Behavior |
|----------------|-------|-------------|
| `ConsoleLog` | `ConsoleLogOperationType` | Logs a message at configurable level (debug/info/warn/error) |
| `ExecuteHttpRequest` | `ExecuteHttpRequestOperationType` | Performs GET/POST/PUT/PATCH/DELETE via `FLEX_HTTP_SERVICE` |
| `ExecuteScript` | `ExecuteScriptOperationType` | Evaluates a user-supplied JavaScript string in a sandboxed context |
| `SetProperty` | `SetPropertyOperationType` | Writes a value to the operation store (`$store` or `$global`) |
| `SetPropertyFromControlProperty` | `SetPropertyFromControlPropertyOperationType` | Reads a DOM element property and writes it to the store |
| `SetFlexControlInput` | `SetFlexControlInputOperationType` | Sets an `@Input()` on a Flex control directly |
| `SetControlVisibility` | `SetControlVisibilityOperationType` | Shows/hides a control by manipulating its DOM `hidden` attribute |
| `SetPropertyFromRouteDataParam` | `SetPropertyFromRouteDataParamOperationType` | Reads a route param (query string / fragment / route param) → writes to store |
| `InvokeMethod` | `InvokeMethodOperationType` | Calls a method on the `IOperationExecutionService` or context |
| `InvokeMethodOnControl` | `InvokeMethodOnControlOperationType` | Calls a method exposed by a Flex control instance |
| `NavigateToRoute` | `NavigateToRouteOperationType` | Navigates the Flex router to a new screen path |
| `Compare` | `CompareOperationType` | Evaluates a condition; branches to success/failure operation list |
| `DisplayAlert` | `DisplayAlertOperationType` | Shows a browser `alert()` dialog |
| `DisplayDialog` | `DisplayDialogOperationType` | Opens a modal using `FLEX_DIALOG_SERVICE` |
| `DisplayComponentDialog` | `DisplayComponentDialogOperationType` | Opens a dynamically-loaded Angular component in a dialog |
| `DisplayToast` | `DisplayToastOperationType` | Shows a success/error/info toast via `FLEX_TOAST_SERVICE` |
| `BindFormData` | `BindFormDataOperationType` | Reads form values and writes them to the operation store |
| `BroadcastMessageToFlexHost` | `BroadcastMessageToFlexHostOperationType` | Sends a message to the parent Flex host frame via `FLEX_BROADCAST_SERVICE` |

### Compare Operation Details

`CompareOperationType` is the only operation with **conditional branching**:

```typescript
interface CompareOperationConfigData {
  conditions: Condition[];
  connector: Connector;       // And | Or
  onTrueOperationListId: string;
  onFalseOperationListId?: string;
}
```

If the condition evaluates to true, it runs `onTrueOperationListId`; if false and `onFalseOperationListId` is set, it runs that list instead. This enables conditional workflows without custom code.

---

## 11. Adding a New Operation Type

1. Create a directory under `src/lib/operations/<operation-name>/`
2. Create `config-data.ts` (the TypeScript interface for the operation's data config)
3. Create `<name>.operation.ts` implementing `IOperationType`:

```typescript
@Injectable()
export class MyOperationType implements IOperationType<MyConfigData> {
  operationType = MyOperationType.OperationTypeName;
  static readonly OperationTypeName = 'MyOperation';

  constructor(@Inject(FLEX_LOG_SERVICE) private logService: ILogService) {}

  async execute(context: IContext, config: MyConfigData, args: IOperationExecutionArgs): Promise<void> {
    this.logService.debug('[my-operation]', 'Executing with config', config);
    // ... implementation
  }
}
```

4. Register in `FlexOperationsModule.providers` via the `OperationTypeRegistryService`
5. Export the type and config interface from `src/index.ts`

---

## 12. Cancellation System

Operations support cooperative cancellation via `OperationCancellationToken`:

```
  OperationCancellationToken
  ├── isCancelled: boolean
  ├── cancel(): void
  └── If isCancelled → executor throws OperationCancellationError

  OperationCancellationError
  └── Caught by OperationExecutorService but re-thrown (not logged/toasted)
  └── Caught by tryExecute() → returns false silently
```

This matters for the `shouldCancelPreviousInvocationsOfList` flag: if a user triggers an operation list twice in quick succession (e.g., rapid search requests), the first invocation's token is cancelled and it stops at the next cancellation check point.

---

## 13. Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Plugin registry** | `OperationTypeRegistryService` decouples runtime from concrete type classes |
| **Sequential pipelines** | Operations run `await`-sequentially to avoid race conditions |
| **Cooperative cancellation** | Token-based cancellation, not `Promise.race` or forced kill |
| **Fail-safe error handling** | Per-operation try-catch: one bad operation doesn't prevent error reporting |
| **Separation of concerns** | Config reading (from `flex-config`), execution (executor), state (NgRx) are all separate |
| **Immutable state** | Immer `produce()` ensures NgRx state immutability |

---

*Generated from codebase analysis — `develop` branch, March 2026.*
