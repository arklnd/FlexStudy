# Full Library Stack — Architecture Deep Dive

> **Audience**: Engineers new to this codebase.  
> **Companion to**: `libs/flex-runtime/ARCHITECTURE.md` (read that first).  
> **Purpose**: Document the architecture of every library under `libs/` — what it owns, what it exports, and how it relates to its neighbours.

---

## Table of Contents

1. [Library Map — 10,000-ft View](#1-library-map--10000-ft-view)
2. [Dependency Hierarchy](#2-dependency-hierarchy)
3. [flex-config — The Schema Dictionary](#3-flex-config--the-schema-dictionary)
4. [flex-types — Pure TypeScript Utilities](#4-flex-types--pure-typescript-utilities)
5. [flex-shared — Contracts & Cross-Cutting Services](#5-flex-shared--contracts--cross-cutting-services)
6. [flex-operations — The Operation Engine](#6-flex-operations--the-operation-engine)
7. [flex-forms — Dynamic Form Subsystem](#7-flex-forms--dynamic-form-subsystem)
8. [flex-router — Screen Navigation](#8-flex-router--screen-navigation)
9. [flex-host — Lightweight Consumer Wrapper](#9-flex-host--lightweight-consumer-wrapper)
10. [onbase-apps-shell — Application Shell UI](#10-onbase-apps-shell--application-shell-ui)
11. [onbase-web-server — Full OnBase Integration Layer](#11-onbase-web-server--full-onbase-integration-layer)
12. [onbase-web-server-core — Low-Level Core Contracts](#12-onbase-web-server-core--low-level-core-contracts)
13. [shared-components — Hyland UI Component Library](#13-shared-components--hyland-ui-component-library)
14. [wv-components — WorkView-Specific Components](#14-wv-components--workview-specific-components)
15. [wf-approval-mgmt-lib — Workflow Approval Management](#15-wf-approval-mgmt-lib--workflow-approval-management)
16. [e2e-shared — Cypress Test Utilities](#16-e2e-shared--cypress-test-utilities)
17. [Cross-Library Data-Flow Walkthrough](#17-cross-library-data-flow-walkthrough)
18. [Design Principles (Repeated Across Libraries)](#18-design-principles-repeated-across-libraries)

---

## 1. Library Map — 10,000-ft View

```
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │                         APPLICATIONS (apps/)                                 │
 │   client-app  |  dev-app  |  onbase-apps-host  |  workflow-approval-mgmt     │
 └──────────────────────────────────┬───────────────────────────────────────────┘
                                    │  consume
           ┌────────────────────────▼──────────────────────────────┐
           │              ORCHESTRATION LAYER                      │
           │   flex-runtime  ◄── bootstraps & wires everything     │
           │   flex-host     ◄── thin wrapper exposing <flex-host> │
           │   onbase-apps-shell ◄── outer shell / nav chrome      │
           └──┬──────────┬──────────────┬──────────────────────────┘
              │          │              │
   ┌──────────▼──┐  ┌────▼────────┐  ┌──▼──────────────────────────┐
   │ flex-router │  │ flex-forms  │  │  flex-operations            │
   │ (navigation)│  │ (forms)     │  │  (all side-effect actions)  │
   └──────────┬──┘  └─────┬───────┘  └──┬──────────────────────────┘
              │           │             │
   ┌──────────▼───────────▼─────────────▼───────────────────────────┐
   │                  FOUNDATION LAYER                              │
   │   flex-config   (shape of every config object)                 │
   │   flex-shared   (DI contracts / service interfaces)            │
   │   flex-types    (TypeScript type utilities)                    │
   └────────────────────────────────────────────────────────────────┘
              │
   ┌──────────▼──────────────────────────────────────────────────────┐
   │               ONBASE INTEGRATION LAYER                          │
   │   onbase-web-server        (viewer, search, workflow, forms)    │
   │   onbase-web-server-core   (low-level tokens & models)          │
   └─────────────────────────────────────────────────────────────────┘
              │
   ┌──────────▼──────────────────────────────────────────────────────┐
   │                  UI COMPONENT LIBRARIES                         │
   │   shared-components   (generic Hyland UI widgets)               │
   │   wv-components       (WorkView-specific widgets)               │
   │   wf-approval-mgmt-lib (Workflow Approval UI + state)           │
   └─────────────────────────────────────────────────────────────────┘
              │
   ┌──────────▼──────────────────────────────────────────────────────┐
   │                   TESTING INFRASTRUCTURE                        │
   │   e2e-shared   (Cypress page-objects & helpers)                 │
   └─────────────────────────────────────────────────────────────────┘
```

---

## 2. Dependency Hierarchy

```
  ┌───────────────────────┬──────────────────────────────────────────────────────────┐
  │ Library               │ Imports from                                             │
  ├───────────────────────┼──────────────────────────────────────────────────────────┤
  │ flex-types            │ (none — zero deps)                                       │
  ├───────────────────────┼──────────────────────────────────────────────────────────┤
  │ flex-config           │ flex-types                                               │
  ├───────────────────────┼──────────────────────────────────────────────────────────┤
  │ flex-shared           │ flex-config, flex-types                                  │
  ├───────────────────────┼──────────────────────────────────────────────────────────┤
  │ flex-operations       │ flex-shared, flex-config, flex-types                     │
  ├───────────────────────┼──────────────────────────────────────────────────────────┤
  │ flex-forms            │ flex-shared, flex-config                                 │
  ├───────────────────────┼──────────────────────────────────────────────────────────┤
  │ flex-router           │ flex-shared, flex-config, flex-types                     │
  ├───────────────────────┼──────────────────────────────────────────────────────────┤
  │ flex-runtime          │ flex-router, flex-forms, flex-operations, flex-config,   │
  │                       │ flex-shared, flex-types                                  │
  ├───────────────────────┼──────────────────────────────────────────────────────────┤
  │ flex-host             │ flex-runtime, flex-router, flex-shared, flex-config      │
  ├───────────────────────┼──────────────────────────────────────────────────────────┤
  │ onbase-web-server-core│ (Angular + Angular Material only)                        │
  ├───────────────────────┼──────────────────────────────────────────────────────────┤
  │ onbase-web-server     │ onbase-web-server-core, flex-shared, shared-components   │
  ├───────────────────────┼──────────────────────────────────────────────────────────┤
  │ onbase-apps-shell     │ flex-runtime, flex-shared, onbase-web-server             │
  ├───────────────────────┼──────────────────────────────────────────────────────────┤
  │ shared-components     │ (Angular + Hyland UI only)                               │
  ├───────────────────────┼──────────────────────────────────────────────────────────┤
  │ wv-components         │ shared-components, flex-shared                           │
  ├───────────────────────┼──────────────────────────────────────────────────────────┤
  │ wf-approval-mgmt-lib  │ shared-components, flex-shared, wv-components            │
  ├───────────────────────┼──────────────────────────────────────────────────────────┤
  │ e2e-shared            │ Cypress only — no app-level imports                      │
  └───────────────────────┴──────────────────────────────────────────────────────────┘
```

**Golden rule**: Lower layers must NEVER import from higher layers. `flex-types` knows nothing about `flex-forms`. `flex-config` knows nothing about `flex-router`. Arrows always point downward.

---

## 3. `flex-config` — The Schema Dictionary

### What it owns

```
  flex-config
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  MODELS  (data shape definitions)                           │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ ApplicationConfiguration   top-level app config      │   │
  │  │ ScreenConfiguration        a single screen's shape   │   │
  │  │ ControlConfiguration       a single control's shape  │   │
  │  │ ControlDefinition          capability declaration    │   │
  │  │ OperationConfiguration     one operation step        │   │
  │  │ RouteConfiguration         URL routing rules         │   │
  │  │ PropertyBindingConfiguration  data binding rules     │   │
  │  │ FlexRuntimeConfiguration   runtime feature flags     │   │
  │  │ EnvironmentVariableCollection  env-specific values   │   │
  │  │ LifecycleHook / LifecycleEvent  hooks on control life│   │
  │  │ Theme / ThemeConfiguration  theming config           │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  CONFIG CREATORS  (fluent builder API)                      │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ FlexApplicationConfigCreator   builds full app cfg   │   │
  │  │ FlexScreenConfigCreator        builds a screen cfg   │   │
  │  │ ControlConfigCreator           builds a control cfg  │   │
  │  │ OperationConfigCreator         builds an op config   │   │
  │  │ OperationListConfigCreator     builds op list        │   │
  │  │ EventMappingConfigCreator      maps events → ops     │   │
  │  │ HooksConfigCreator             lifecycle hooks       │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  HELPERS:  constant(), property(), storeGloballyAs()        │
  └─────────────────────────────────────────────────────────────┘
```

### Key design decision

`flex-config` holds **only data shapes** — interfaces and plain-object creators. It has zero Angular dependency and zero runtime logic. This makes it safe to import from non-Angular environments (e.g., config tooling, tests that build configs without Angular).

### Config creator example flow

```
  FlexApplicationConfigCreator
    │
    ├─ .addScreen( FlexScreenConfigCreator )
    │        │
    │        ├─ .addControl( ControlConfigCreator )
    │        │        │
    │        │        └─ .addEventMapping( EventMappingConfigCreator )
    │        │                 │
    │        │                 └─ .addOperation( OperationConfigCreator )
    │        │
    │        └─ .addRoute( RouteConfiguration )
    │
    └─ .build()  →  ApplicationConfiguration  (plain object)
```

---

## 4. `flex-types` — Pure TypeScript Utilities

### What it owns

```
  flex-types/src/lib/
  ┌─────────────────────────────────────────────────────────┐
  │  extract-emitted-type.type.ts    EventEmitter<T> → T    │
  │  omit-function-props.type.ts     remove func members    │
  │  omit-props-of-type.type.ts      omit by value type     │
  │  pick-observable-props.type.ts   pick Observable<T>s    │
  │  pick-props-of-type.type.ts      pick by value type     │
  │  typed-simple-changes.type.ts    typed ngOnChanges      │
  │  union.type.ts                   Union helper type      │
  └─────────────────────────────────────────────────────────┘
```

### Why it exists

These generics are used across **every** other library to write type-safe code without repeating boilerplate. They carry zero runtime weight — they are erased at compile time.

```typescript
  // Example: typed SimpleChanges — avoids casting
  ngOnChanges(changes: TypedSimpleChanges<MyComponent>) {
    if (changes.myInput?.currentValue) { ... }
  }
```

---

## 5. `flex-shared` — Contracts & Cross-Cutting Services

### What it owns

`flex-shared` is the **interface registry** of the Flex system. It defines the DI tokens every library uses to communicate without hard dependencies on each other.

```
  flex-shared
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  SERVICE INTERFACE TOKENS                                   │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ FLEX_AUTHENTICATION_SERVICE  / IAuthService          │   │
  │  │ FLEX_BROADCAST_SERVICE       / IBroadcastService     │   │
  │  │ FLEX_CONFIG_SERVICE          / IConfigurationService │   │
  │  │ FLEX_DIALOG_SERVICE          / IDialogService        │   │
  │  │ ENVIRONMENT_VARIABLES_SERVICE/ IEnvVarsService       │   │
  │  │ EXPRESSION_EVALUATOR_SERVICE / IExpressionEvaluator  │   │
  │  │ FLEX_HTTP_SERVICE            / IHttpService          │   │
  │  │ FLEX_LOG_SERVICE             / ILogService           │   │
  │  │ OPERATION_EXECUTION_SERVICE  / IOperationExecution   │   │
  │  │ OPERATION_EXECUTOR_SERVICE   / IOperationExecutor    │   │
  │  │ FLEX_PROPERTY_BINDING_SERVICE/ IPropertyBinding      │   │
  │  │ FLEX_ROUTER_PARAMETER_SERVICE/ IRouterParamService   │   │
  │  │ FLEX_RUNTIME_CONFIG_SERVICE  / IFlexRuntimeConfig    │   │
  │  │ FLEX_TOAST_SERVICE           / IToastService         │   │
  │  │ COMPONENT_TRACKING_SERVICE   / IComponentTracking    │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  CONCRETE (small) IMPLEMENTATIONS                           │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ HttpService         wraps Angular HttpClient         │   │
  │  │ FlexLogService      console + remote logging         │   │
  │  │ ExpressionEvaluatorService  evaluates expressions    │   │
  │  │ OperationStateStore         NgRx state proxy         │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  UTILITIES                                                  │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ TaskQueue          serial async task scheduler       │   │
  │  │ stringFormat       printf-style string formatting    │   │
  │  │ noopAction         empty NgRx action helper          │   │
  │  │ FlexConstants      global string constants           │   │
  │  │ FlexStyleHelper    component for CSS variable tricks │   │
  │  └──────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────┘
```

### Why everything depends on flex-shared

Every library that needs to call a service uses the token from `flex-shared`. The concrete implementation is registered at the module level (usually in `flex-runtime`'s `FlexModule.forRoot()`). This enforces Inversion of Control across the entire stack.

```
  flex-forms needs to log an error:
    ↓ injects FLEX_LOG_SERVICE (from flex-shared)
    ↓ does NOT import FlexLogService directly
    ↓ the actual logger is decided by the host app / test harness
```

---

## 6. `flex-operations` — The Operation Engine

### What it owns

```
  flex-operations
  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │  OPERATION TYPES  (each folder = one operation)              │
  │  ┌────────────────────────────────────────────────────────┐  │
  │  │ bind-form-data           push form values to state     │  │
  │  │ broadcast-message-to-flex-host  cross-frame messaging  │  │
  │  │ compare                  conditional branching logic   │  │
  │  │ console-log              debug logging                 │  │
  │  │ display-alert            modal alert dialogs           │  │
  │  │ display-component-dialog render Angular component as   │  │
  │  │                          a dialog                      │  │
  │  │ display-dialog           show a Flex screen in dialog  │  │
  │  │ display-toast            snackbar toasts               │  │
  │  │ execute-http-request     arbitrary REST calls          │  │
  │  │ execute-script           run inline JS/expression      │  │
  │  │ invoke-method            call a named method on a ctrl │  │
  │  │ invoke-method-on-control target a specific control     │  │
  │  │ navigate-to-route        push Angular router URL       │  │
  │  │ set-control-visibility   show/hide UI controls         │  │
  │  │ set-flex-control-input   push value into ctrl @Input   │  │
  │  │ set-property             write to state store          │  │
  │  │ set-property-from-control-property  copy ctrl → state  │  │
  │  │ set-property-from-route-data-param  copy URL → state   │  │
  │  └────────────────────────────────────────────────────────┘  │
  │                                                              │
  │  STATE STORE                                                 │
  │  ┌────────────────────────────────────────────────────────┐  │
  │  │ OperationStore            NgRx feature slice           │  │
  │  │ OperationStateStoreProxy  wraps Store, adds helpers    │  │
  │  └────────────────────────────────────────────────────────┘  │
  │                                                              │
  │  SERVICES                                                    │
  │  ┌────────────────────────────────────────────────────────┐  │
  │  │ OperationExecutionService  resolves & runs op chain    │  │
  │  │ OperationExecutorService   resolves a single op type   │  │
  │  │ OperationTypeRegistry      maps type string → class    │  │
  │  │ ExpressionEvaluator        evaluates binding exprs     │  │
  │  │ SessionBasedLoggerService  per-session log correlation │  │
  │  └────────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────┘
```

### Operation execution pipeline

```
  User Event (e.g., button click)
    │
    ▼
  EventMapping in ControlConfiguration
  (maps "click" → [OperationListConfiguration])
    │
    ▼
  OperationExecutionService.execute(operationList)
    │
    ├─ resolves each OperationConfiguration.type string
    │  via OperationTypeRegistry
    │        │
    │        └─ returns concrete IOperationType handler
    │
    ├─ evaluates expression bindings in config data
    │  via ExpressionEvaluatorService
    │
    └─ calls IOperationType.execute(context, configData)
              │
              ├─ may dispatch NgRx action
              ├─ may call IHttpService
              ├─ may navigate via IFlexRouterService
              └─ returns Observable<void>
```

### Every operation is a class

Each operation is a self-contained class implementing `IOperationType`:

```
  IOperationType
    execute(context: IContext, configData: T): Observable<void>
```

Adding a new operation = add a new folder, implement `IOperationType`, register in `OperationTypeRegistry`. No changes to `OperationExecutionService`.

---

## 7. `flex-forms` — Dynamic Form Subsystem

### What it owns

```
  flex-forms
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  FormElementFacade                                          │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ Bridges a dynamic control ↔ Angular Reactive Form    │   │
  │  │ Reads FormField config from flex-config              │   │
  │  │ Creates AbstractControl instances dynamically        │   │
  │  │ Wires validators from ValidatorRegistry              │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  FormScreenBinding                                          │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ Associates a FormGroup with a screenInstanceId       │   │
  │  │ Registered in FormToScreenRegistryService            │   │
  │  │ Used by flex-runtime's dirty-check resolver          │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  FormSubmissionService                                      │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ Triggers form validation before submit operation     │   │
  │  │ Returns Observable<boolean> (valid or not)           │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  FormFieldErrorService                                      │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ Centralises error-message resolution per field       │   │
  │  │ Maps ValidatorError keys → localised strings         │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  Validators (validation-types/)                             │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ Required, MinLength, MaxLength, Pattern, Email,      │   │
  │  │ Min, Max, Custom expression validators, etc.         │   │
  │  │ All registered  in validation-types-list.ts          │   │
  │  └──────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────┘
```

### Dirty-check integration with flex-runtime

```
  flex-forms creates FormGroup for screen [id: "screen-42"]
       │
       └─ registers FormScreenBinding(screenId="screen-42", group)

  flex-runtime's DoesScreenContainDirtyFormResolver
       │
       └─ asks FormToScreenRegistryService:
              "is any form for 'screen-42' dirty?"
                      │
              YES → block navigation
              NO  → allow navigation
```

---

## 8. `flex-router` — Screen Navigation

### What it owns

```
  flex-router
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  COMPONENTS                                                 │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ FlexScreenComponent      renders a single screen     │   │
  │  │   • parses ScreenConfiguration                       │   │
  │  │   • dynamically instantiates controls into DOM       │   │
  │  │ FlexRouterOutletComponent  named outlet placeholder  │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  DIRECTIVES                                                 │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ FlexRouterLink         declarative navigation        │   │
  │  │ FlexRouterLinkActive   CSS class when route active   │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  SERVICES                                                   │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ FlexRouterService          imperative navigation     │   │
  │  │ FlexRouterNavigationFacilitatorService               │   │
  │  │                            full nav lifecycle mgmt   │   │
  │  │ RouterParameterService     reads :param & ?query     │   │
  │  │ FocusService               manages focus on nav      │   │
  │  │ ScreenTrackingService      tracks mounted screens    │   │
  │  │ AbstractScreenService      base for custom screens   │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  INTERFACES / TOKENS                                        │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ FLEX_ROUTER_SERVICE         injectable token         │   │
  │  │ FLEX_ROUTER_CONFIG_SERVICE  route registration token │   │
  │  │ FLEX_ROUTE_GUARDS_TOKEN     pluggable guards         │   │
  │  │ SCREEN_SERVICE              screen info provider     │   │
  │  │ CAN_NAVIGATE_AWAY_FROM_SCREEN_RESOLVERS              │   │
  │  └──────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────┘
```

### Screen render lifecycle

```
  flex-runtime calls screenControl.setScreenInfo(screenConfig)
       │
       ▼
  FlexRouterService resolves the route
       │
       ▼
  FlexScreenComponent.ngOnInit()
       │
       ├─ reads ScreenConfiguration.controls[]
       ├─ for each ControlConfiguration:
       │     ├─ resolves component class from ControlDefinition
       │     ├─ creates ComponentRef dynamically
       │     ├─ sets @Input bindings from PropertyBindingConfiguration
       │     └─ subscribes to @Output events → OperationExecutionService
       │
       └─ appends all ComponentRefs to host element
```

### Pluggable guards via token

```
  FLEX_ROUTE_GUARDS_TOKEN = {
    canActivate:   [...],   // injected by host app
    canDeactivate: [CanScreenDeactivateGuard],  // provided by flex-host
  }
```

Host apps can add their own route guards by overriding this token — without modifying `flex-router`.

---

## 9. `flex-host` — Lightweight Consumer Wrapper

### What it owns

```
  flex-host
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  FlexHostComponent                                          │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ selector: flex-host                                  │   │
  │  │ @Input() appConfig: FlexRuntimeConfiguration         │   │
  │  │                                                      │   │
  │  │ wraps <flex-app-viewer> (from flex-runtime)          │   │
  │  │ provides CanScreenDeactivateGuard                    │   │
  │  │   via FLEX_ROUTE_GUARDS_TOKEN                        │   │
  │  │ provides PathLocationStrategy                        │   │
  │  └──────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────┘
```

### Why flex-host exists

`flex-runtime` exposes `<flex-app-viewer>` which requires manual DI setup. `flex-host` wraps it with sensible defaults (path-based routing, deactivate guard) so consumer apps embed just:

```html
<flex-host [appConfig]="myConfig"></flex-host>
```

```
  Consumer App Template
    │
    └─ <flex-host [appConfig]="cfg">
              │
              └─ <flex-app-viewer>           (flex-runtime)
                        │
                        └─ <flex-screen>     (flex-router)
                                  │
                                  └─ dynamic controls
```

---

## 10. `onbase-apps-shell` — Application Shell UI

### What it owns

```
  onbase-apps-shell
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  OBAShellComponent                                          │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ The outer chrome of the application:                 │   │
  │  │   • Application header bar                           │   │
  │  │   • Navigation tabs / side-rail                      │   │
  │  │   • Shell-level theming container                    │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  OBAShellViewComponent                                      │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ Content viewport area within the shell               │   │
  │  │ Where <flex-host> / <router-outlet> lives            │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  OBAShellService                                            │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ Controls shell-level state:                          │   │
  │  │   • active application context                       │   │
  │  │   • header title & actions                           │   │
  │  │   • loading indicators                               │   │
  │  └──────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────┘
```

### Shell composition model

```
  <oba-shell>                              ← OBAShellComponent
    <oba-shell-header>                     ← header bar
    <oba-shell-view>                       ← OBAShellViewComponent
      <router-outlet>  or  <flex-host>     ← actual app content
    </oba-shell-view>
  </oba-shell>
```

---

## 11. `onbase-web-server` — Full OnBase Integration Layer

### What it owns

This is the **largest** library — it integrates with all OnBase Web Server capabilities.

```
  onbase-web-server
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  SOURCE VIEWS  (top-level feature modules)                  │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ doc-query/           Document keyword query results  │   │
  │  │ folder-query/        Folder query results            │   │
  │  │ workflow/            Workflow queue & item viewer    │   │
  │  │ workview/            WorkView object list            │   │
  │  │ workview-single-object  Single WV object view        │   │
  │  │ error/               Error screen                    │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  MODULES (functional groups)                                │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ content/                                             │   │
  │  │   viewer/              ViewerComponent               │   │
  │  │   folders/             FolderViewerComponent         │   │
  │  │   data-grid/           ContentDataGridComponent      │   │
  │  │   searching/           SearchPanelComponent          │   │
  │  │   workitem-viewer/     WorkItemViewerComponent       │   │
  │  │   related-items/       Source item relationships     │   │
  │  │   common/              Shared queries, menus         │   │
  │  │                                                      │   │
  │  │ file-upload-enhanced/  FileUploadEnhancedComponent   │   │
  │  │ reporting-dashboards/  DashboardViewerComponent      │   │
  │  │ source-query/          SourceQueryService            │   │
  │  │ unity-forms/           CreateUnityFormComponent      │   │
  │  │ workflow/              WorkflowItemViewerComponent   │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  SHARED INFRASTRUCTURE                                      │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ OnBaseWebServerModule          top-level NgModule    │   │
  │  │ OnBaseWebServerServicesModule  services-only module  │   │
  │  │ OnBaseWebServerDialogService   fullscreen dialogs    │   │
  │  │ BlockingService                UI blocking overlay   │   │
  │  │ SourceChangeNotifierService    notifies on src change│   │
  │  │ WebServerSourceDirective       [src] with auth token │   │
  │  └──────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────┘
```

### Feature module interaction

```
  Host App
    │
    └─ imports OnBaseWebServerModule
              │
              ├─ ViewerComponent          renders documents
              ├─ WorkflowItemViewerComponent  tasks & queues
              ├─ SearchPanelComponent     keyword search
              ├─ FolderViewerComponent    folder hierarchy
              └─ DashboardViewerComponent reporting dashboards

  Each component:
    ├─ receives ScreenConfiguration via @Input bindings
    │    provided by flex-router property binding system
    ├─ calls OnBaseWebServerDialogService for fullscreen views
    ├─ emits events → OperationExecutionService (flex-operations)
    └─ uses ContentQueryService / WorkflowQueryService for data
```

---

## 12. `onbase-web-server-core` — Low-Level Core Contracts

### What it owns

```
  onbase-web-server-core
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  MODELS                                                     │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ WorkItemType   enum of workflow item types           │   │
  │  │ ViewerType     enum of supported viewer types        │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  SHARED TOKENS & INTERFACES                                 │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ WEB_CLIENT_WINDOW / WebClientWindow                  │   │
  │  │     Typed reference to the host web client window    │   │
  │  │ WEB_CLIENT_APP_HOST_CONFIG / WebClientAppHostConfig  │   │
  │  │     Base URL, auth endpoint, feature flags           │   │
  │  │ OB_STORAGE / IOBStorageService                       │   │
  │  │     Abstraction over localStorage / sessionStorage   │   │
  │  │ SESSION_CHECKER / ISessionCheckerService             │   │
  │  │     Checks whether the OBS session is still valid    │   │
  │  │ UNLOAD_MANAGER / IUnloadManagerService               │   │
  │  │     Registers cleanup callbacks for page unload      │   │
  │  │ PAGE / IPageService                                  │   │
  │  │     window.location / reload coordination            │   │
  │  │ DATA_VALIDATION                                      │   │
  │  │     Pluggable data validation layer                  │   │
  │  │ WINDOWS_MANAGER                                      │   │
  │  │     Manages pop-out/secondary window lifecycle       │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  PIPES                                                      │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ TranslationPipe   |  translate  | i18n pipe          │   │
  │  └──────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────┘
```

### Relationship to onbase-web-server

`onbase-web-server-core` is the **stable API** that `onbase-web-server` builds on. Downstream consumers should import tokens from `core`; the full `onbase-web-server` library is needed only when you need actual UI components.

```
  onbase-web-server-core  (tokens, models, pipe)
         ▲
         │  imports only core contracts
  onbase-web-server       (full UI components + services)
```

---

## 13. `shared-components` — Hyland UI Component Library

### What it owns

A library of **presentational** Angular components that implement the Hyland Design System.

```
  shared-components/src/lib/
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  Layout & Structure                                         │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ hy-ng-layout         flex/grid layout wrapper        │   │
  │  │ hy-ng-layout-behavior  responsive breakpoint logic   │   │
  │  │ hy-ng-splitter       resizable split panes           │   │
  │  │ hy-ng-master-detail  master/detail two-column layout │   │
  │  │ hy-ng-spacer         layout gap utility              │   │
  │  │ hy-ng-divider        horizontal/vertical separator   │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  Navigation & Toolbars                                      │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ hy-ng-toolbar        top action bar                  │   │
  │  │ collapsible-toolbar  auto-overflow toolbar           │   │
  │  │ hy-ng-tabs           tab group with routing support  │   │
  │  │ menu-with-buttons    dropdown + direct action combos │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  Content Display                                            │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ hy-ng-card           material card wrapper           │   │
  │  │ hy-ng-text           styled text with variants       │   │
  │  │ hy-ng-data-grid      configurable data grid          │   │
  │  │ hy-ng-expansion-panel accordion section              │   │
  │  │ hy-ng-table-group-menu  group-by menu for grids      │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  User Interaction                                           │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ hy-ng-button         styled primary/secondary btn    │   │
  │  │ hy-ng-search-bar     search input with clear/submit  │   │
  │  │ hy-ng-info-dialog    info/confirmation dialog        │   │
  │  │ hy-ng-progress-spinner  loading indicators           │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  Theming                                                    │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ hy-ng-theme-toggle   light/dark mode switcher        │   │
  │  │ hy-ng-style-helper   CSS custom property injector    │   │
  │  │ theming.scss         SCSS theming entry point        │   │
  │  └──────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────┘
```

### Design principle

All components are **dumb** (presentational only). They receive data through `@Input()` and emit events through `@Output()`. They never inject business services or dispatch NgRx actions directly.

---

## 14. `wv-components` — WorkView-Specific Components

### What it owns

```
  wv-components/src/lib/
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  Data Display                                               │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ hy-ng-workview-object-viewer  renders a WV object    │   │
  │  │ hy-ng-column                 typed WV column def     │   │
  │  │ hy-ng-row                    WV data row wrapper     │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  Form & Filter                                              │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ filter-bar       filter strip for WV queries         │   │
  │  │ form-fields      WV-specific form field controls     │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  Navigation                                                 │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ hy-ng-nav-list   sidebar navigation list for WV      │   │
  │  │ hy-ng-expander   collapsible content expander        │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  Theming & Styling                                          │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ theming.scss              WV-specific SCSS           │   │
  │  │ extend-darkmode.function.scss  dark mode overrides   │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  Pipes                                                      │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ Various display-transformation pipes for WV data     │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  Constraints                                                │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ WorkView field-level constraint definitions          │   │
  │  └──────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────┘
```

`wv-components` extends `shared-components` with WorkView domain-specific widgets. It adds a dark-mode SCSS extension function so WV components participate in the global theming system.

---

## 15. `wf-approval-mgmt-lib` — Workflow Approval Management

### What it owns

```
  wf-approval-mgmt-lib
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  STATE (NgRx facades)                                       │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ ApprovalProcessesFacadeService                       │   │
  │  │     list of approval processes + pagination          │   │
  │  │ ApprovalProcessDetailsFacadeService                  │   │
  │  │     full detail of a single approval process         │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  ALERT COMPONENTS                                           │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ HyNgModalComponent        generic confirmation modal │   │
  │  │ HyNgSimpleAlertComponent  inline alert banner        │   │
  │  │ HyNgRemoveAlertComponent  destructive action confirm │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  SHARED COMPONENTS                                          │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ HyNgConditionsComponent     approval rule conditions │   │
  │  │ HyNgReplaceApproverComponent replace an approver     │   │
  │  │ HyNgImportCsvComponent      CSV import for approval  │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  SERVICES                                                   │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ SnackbarService   toast notifications for approvals  │   │
  │  │ ProcessDataService  raw HTTP data for processes      │   │
  │  └──────────────────────────────────────────────────────┘   │
  │                                                             │
  │  i18n                                                       │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ assets/i18n/en-us.ts  English translation strings    │   │
  │  └──────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────┘
```

### Facade pattern

```
  Component (HyNgConditionsComponent)
    │
    └─ injects ApprovalProcessesFacadeService
              │
              ├─ .processes$     Observable<ApprovalProcess[]>
              ├─ .loading$       Observable<boolean>
              └─ .loadProcesses() → dispatches NgRx action
                        │
                        ▼
               NgRx Effect → ProcessDataService → HTTP → Store
                        │
                        ▼
               .processes$ emits updated data → component re-renders
```

---

## 16. `e2e-shared` — Cypress Test Utilities

### What it owns

```
  e2e-shared/src/lib/cypress/
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  Shared Cypress page-object helpers and utility functions   │
  │  imported by:                                               │
  │    • apps/client-app-e2e                                    │
  │    • apps/onbase-apps-host-e2e                              │
  │    • apps/workflow-approval-mgmt-e2e                        │
  │                                                             │
  │  Typical contents:                                          │
  │  ┌──────────────────────────────────────────────────────┐   │
  │  │ Base page-object classes                             │   │
  │  │ Custom Cypress commands shared across test suites    │   │
  │  │ Test data helpers & factory functions                │   │
  │  └──────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────┘
```

`e2e-shared` has **no production build output** — it is test-only and is never imported by application or library code.

---

## 17. Cross-Library Data-Flow Walkthrough

The following traces a user clicking a button that runs a `navigate-to-route` operation.

```
  ┌─────────────────────────────────────────────────────────────────────────┐
  │  STEP 1 — Config loading (startup)                                      │
  │                                                                         │
  │  flex-config shapes the config:                                         │
  │    ApplicationConfiguration.screens[0].controls[0].eventMappings        │
  │      .click → OperationListConfiguration                                │
  │        [0] → NavigateToRouteOperationConfiguration { route: '/detail' } │
  └─────────────────────────────────────────────────────────────────────────┘
              │
              ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │  STEP 2 — Screen renders (flex-router)                                  │
  │                                                                         │
  │  FlexScreenComponent reads control config                               │
  │  instantiates ButtonComponent dynamically                               │
  │  subscribes to ButtonComponent's (click) @Output                        │
  └─────────────────────────────────────────────────────────────────────────┘
              │
              ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │  STEP 3 — User clicks the button                                        │
  │                                                                         │
  │  ButtonComponent emits (click)                                          │
  │  FlexScreenComponent receives → calls OperationExecutionService.execute │
  └─────────────────────────────────────────────────────────────────────────┘
              │
              ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │  STEP 4 — Operation pipeline (flex-operations)                          │
  │                                                                         │
  │  OperationExecutionService                                              │
  │    → ExpressionEvaluatorService resolves dynamic expressions            │
  │    → OperationTypeRegistry.resolve('navigate-to-route')                 │
  │    → NavigateToRouteOperationType.execute(context, { route: '/detail' })│
  │          → injects FLEX_ROUTER_SERVICE (token from flex-shared)         │
  │          → calls IFlexRouterService.navigate('/detail')                 │
  └─────────────────────────────────────────────────────────────────────────┘
              │
              ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │  STEP 5 — Route change (flex-router)                                    │
  │                                                                         │
  │  FlexRouterService delegates to Angular Router                          │
  │  DoesScreenContainDirtyFormResolver (flex-runtime)                      │
  │    → asks FormToScreenRegistryService (flex-forms): any dirty forms?    │
  │    → NO → navigation proceeds                                           │
  │  FlexScreenComponent for '/detail' mounts                               │
  └─────────────────────────────────────────────────────────────────────────┘
```

---

## 18. Design Principles (Repeated Across Libraries)

```
  ┌────────────────────────────────────────────────────────────────────────┐
  │  PRINCIPLE                  HOW IT APPEARS                             │
  ├────────────────────────────────────────────────────────────────────────┤
  │  One library = one concern  flex-config = types only                   │
  │                             flex-operations = operations only          │
  │                             shared-components = UI only                │
  ├────────────────────────────────────────────────────────────────────────┤
  │  Depend downward only       Higher layers import lower layers;         │
  │                             lower layers NEVER import higher.          │
  ├────────────────────────────────────────────────────────────────────────┤
  │  Inject, don't instantiate  Every cross-library call goes through a    │
  │                             DI token defined in flex-shared.           │
  ├────────────────────────────────────────────────────────────────────────┤
  │  Dumb components            UI components (shared-components,          │
  │                             wv-components) are presentational.         │
  │                             Business logic lives in services/facades.  │
  ├────────────────────────────────────────────────────────────────────────┤
  │  Configuration-driven       Runtime behaviour is controlled by         │
  │                             ApplicationConfiguration from backend —    │
  │                             no feature is hard-coded in templates.     │
  ├────────────────────────────────────────────────────────────────────────┤
  │  State-driven               All async side-effects go through NgRx     │
  │     (NgRx everywhere)       stores/effects. Components never call      │
  │                             HTTP services directly.                    │
  ├────────────────────────────────────────────────────────────────────────┤
  │  Pluggable operations       New operations = new class implementing    │
  │                             IOperationType. No changes to the engine.  │
  └────────────────────────────────────────────────────────────────────────┘
```

---

*Generated from codebase analysis — `develop` branch, 4th March 2026.*
