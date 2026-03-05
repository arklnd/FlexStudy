# `flex-config` — Architecture Deep Dive

> **Audience**: Engineers writing Flex application configurations or building tooling that generates them.  
> **Package**: `@hyland/flex-config`  
> **LOC**: 1,059 lines (TS src) · 1 spec · 0 HTML · 0 SCSS  
> **Usage score**: 108 import-lines across the repo · Tier 1 — CANNOT REMOVE  
> **Companion document**: See [`libs/ARCHITECTURE.md`](../ARCHITECTURE.md) for the full cross-library picture.

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [The Big Picture](#2-the-big-picture)
3. [Two-Layer Design](#3-two-layer-design)
4. [Data Models Layer — All Types](#4-data-models-layer--all-types)
   - [Root Configuration Objects](#41-root-configuration-objects)
   - [Control Configuration](#42-control-configuration)
   - [Control Definitions](#43-control-definitions)
   - [Operations](#44-operations)
   - [Routing](#45-routing)
   - [Forms](#46-forms)
   - [Lifecycle Hooks](#47-lifecycle-hooks)
   - [Theming](#48-theming)
5. [Creator API (Fluent Builder) Layer](#5-creator-api-fluent-builder-layer)
   - [FlexApplicationConfigCreator](#51-flexapplicationconfigcreator)
   - [FlexScreenConfigCreator](#52-flexscreenconfigcreator)
   - [ControlConfigCreator](#53-controlconfigcreator)
   - [OperationListConfigCreator + OperationConfigCreator](#54-operationlistconfigcreator--operationconfigcreator)
   - [HooksConfigCreator](#55-hooksconfigcreator)
   - [EventMappingConfigCreator](#56-eventmappingconfigcreator)
   - [Config Creator Helpers](#57-config-creator-helpers)
6. [Full FlexRuntimeConfiguration Shape](#6-full-flexruntimeconfiguration-shape)
7. [Design Principles](#7-design-principles)

---

## 1. Purpose

`flex-config` is the **pure data contract library** for the Flex runtime. It owns two things:

1. **Data model TypeScript interfaces** — the exact shapes of every JSON object that the Flex runtime consumes at startup to know which screens to render, which components to mount, which operations to execute, and how routing should behave.

2. **Fluent creator API** — a set of builder classes that make it easy to construct valid `FlexRuntimeConfiguration` objects in TypeScript code (for tests, dev-app setup, and feature configuration).

This library has **zero runtime code**. It contains no Angular modules, no services, and no DI tokens. It is the shared vocabulary between the libraries that produce configurations (`onbase-apps-host`, `client-app`) and the libraries that consume them (`flex-runtime`, `flex-router`, `flex-operations`).

---

## 2. The Big Picture

```
  Configuration Authoring                      Configuration Consumption
  ┌──────────────────────────────────┐         ┌────────────────────────────────────────┐
  │  FlexApplicationConfigCreator    │         │  flex-runtime                          │
  │  .createScreen()                 │──────►  │  reads FlexRuntimeConfiguration        │
  │    .addControlConfiguration()    │         │  bootstraps screens, mounts controls   │
  │    .addOperationList()           │  flex-  ├────────────────────────────────────────┤
  │    .addHook()                    │ config  │  flex-router                           │
  │  .toConfig()                     │  types  │  reads RouteConfiguration, breakpoints │
  │                                  │         ├────────────────────────────────────────┤
  │  or: build raw JSON matching     │         │  flex-operations                       │
  │  the TypeScript data shapes      │         │  reads OperationConfiguration,         │
  └──────────────────────────────────┘         │  OperationListConfiguration            │
                                               ├────────────────────────────────────────┤
                                               │  flex-forms                            │
                                               │  reads FormField, FormFieldValidation  │
                                               └────────────────────────────────────────┘
```

---

## 3. Two-Layer Design

```
  flex-config/src/lib/
  ├── application-configuration/          ← DATA MODELS (pure interfaces/types)
  │   ├── application-configuration.model.ts
  │   ├── flex-runtime-configuration.model.ts
  │   ├── screen-configuration.model.ts
  │   ├── shell-configuration.model.ts
  │   ├── control-configuration/          ← control instance data
  │   ├── control-definition/             ← reusable control metadata
  │   ├── operations/                     ← operation config shapes
  │   ├── routing/                        ← route configuration
  │   └── forms/                          ← form field + validation
  │
  └── application-configuration.creator/  ← FLUENT BUILDER API
      ├── flex-application.config-creator.ts   FlexApplicationConfigCreator
      ├── flex-screen.config-creator.ts         FlexScreenConfigCreator
      ├── control.config-creator.ts             ControlConfigCreator<T>
      ├── operation.config-creator.ts           OperationConfigCreator
      ├── operation-list.config-creator.ts      OperationListConfigCreator
      ├── hooks.config-creator.ts               HooksConfigCreator
      ├── event-mapping.config-creator.ts       EventMappingConfigCreator
      └── config-creator-helpers.ts             property(), constant(), storeGloballyAs(), storeLocallyAs()
```

---

## 4. Data Models Layer — All Types

### 4.1 Root Configuration Objects

#### `FlexRuntimeConfiguration`
The top-level object passed to `FlexHostComponent.appConfig`. Contains the entire application definition.

```typescript
interface FlexRuntimeConfiguration {
  config: ApplicationConfiguration;
  envData: EnvironmentVariableCollection;
  minimumLoggingLevel: HyLogLevel;
  shellConfig: ShellConfiguration;
  stateActions: FlexStateAction[];
  onBaseSettings: { contentIntelligenceConnector: string };
}
```

#### `ApplicationConfiguration`
The config sub-object inside `FlexRuntimeConfiguration.config`:

```typescript
interface ApplicationConfiguration {
  bootstrapScreen: string;        // screen ID to navigate to on startup
  controlDefinitions: any[];      // reusable control class registrations
  screens: ScreenConfiguration[];
  route: string;                  // base URL route for this flex app
}
```

#### `ScreenConfiguration`
Describes a single routable screen:

```typescript
interface ScreenConfiguration {
  id: string;
  systemName: string;
  htmlContentTemplate: string;     // HTML string defining the layout
  hooks: LifecycleHook[];
  operationLists: OperationListConfiguration[];
  controlConfigurations: ControlConfiguration[];
  routes: RouteConfiguration[];
}
```

---

### 4.2 Control Configuration

A **control configuration** is the per-instance settings for a component placed on a screen:

```typescript
interface ControlConfiguration {
  id: string;
  selector: string;
  styles: FlexSupportedStylesType;
  data: ControlConfigurationData[];       // property bindings
  formData: ControlConfigurationFormData[];
  inboundEvents: EventMapping[];
  outboundEvents: EventMapping[];
}
```

**`PropertyBindingConfiguration`** — binds an Angular `@Input()` to a data source:

```typescript
interface PropertyBindingConfiguration {
  property: string;               // the @Input() name
  value: ValueConfigData;         // constant, property ref, or store ref
}
```

**`PropertyBindingType`**: `Constant | Property | StoreValue | GlobalStoreValue`

---

### 4.3 Control Definitions

A **control definition** is the reusable metadata about a custom control class (its registered properties, events, and methods):

```typescript
interface ControlDefinition {
  selector: string;
  properties: ControlPropertyDefinition[];
  inboundEvents: ControlInboundEventDefinition[];
  outboundEvents: ControlOutboundEventDefinition[];
  methods: ControlMethodDefinition[];
}
```

---

### 4.4 Operations

**`OperationListConfiguration`** — a named list of sequential operations:

```typescript
interface OperationListConfiguration {
  id: string;
  systemName: string;
  shouldCancelPreviousInvocationsOfList: boolean;
  operations: OperationConfiguration[];
  eventMappings: OperationEventMapping[];
}
```

**`OperationConfiguration`** — a single operation step:

```typescript
interface OperationConfiguration {
  operationType: string;
  data: OperationConfigData;
}
```

**`OperationEventMapping`** — wires a control's `@Output()` event to an operation list:

```typescript
interface OperationEventMapping {
  controlId: string;
  outboundEventName: string;
  condition?: ValueConfigData;
}
```

**`ValueConfigData<T, U>`** — a typed value reference, discriminated by `OperationTypeToValueType`:

| Type | Meaning |
|------|---------|
| `Constant` | A literal value baked into the config |
| `Property` | A reference to a store property (`$store.name`) |
| `GlobalStoreValue` | Reference to the global store (`$global.name`) |

**`StoreValueConfigData`** — describes where to write an operation's output:

```typescript
interface StoreValueConfigData {
  toProperty: string;
  global: boolean;      // true = global store, false = local (operation list scope)
}
```

---

### 4.5 Routing

**`RouteConfiguration`** — maps a URL path to a screen:

```typescript
interface RouteConfiguration {
  path: string;
  screenId: string;
  breakpoints: RouteBreakpointConfiguration[];
}
```

**`RouteBreakpointConfiguration`** — maps a CSS-style breakpoint to the router outlet that should host the screen:

```typescript
interface RouteBreakpointConfiguration {
  breakpoint: string;               // CSS media query width or '*' for default
  flexRouterOutletControlId: string;
}
```

---

### 4.6 Forms

**`FormField`** — defines a single form input field with validation rules:

```typescript
interface FormField {
  id: string;
  defaultValue?: any;
  validations?: FormFieldValidation[];
}
```

**`FormFieldValidation`** — a validation rule:

```typescript
interface FormFieldValidation {
  type: string;             // e.g. 'required', 'min', 'maxLength'
  data?: FormFieldValidationData;
  errorMessageKey: string;
}
```

**`FormValidationUpdateOn`**: `change | blur | submit`

---

### 4.7 Lifecycle Hooks

**`LifecycleHook`** — binds a lifecycle event to an operation list:

```typescript
interface LifecycleHook {
  event: LifecycleEvent;
  operationListId: string;
}
```

**`LifecycleEvent`**: `ScreenInit | ScreenDestroy | ControlInit | ControlInputChanges | GlobalStoreValueChanged | RouteParamChanged`

**`GlobalStoreValueChangedHooks`** / **`RouteParamChangedHooks`** — extended hook types that fire on specific property/param name changes.

---

### 4.8 Theming

**`ThemeConfiguration`**:

```typescript
interface ThemeConfiguration {
  cssVariables: CSSVariable[];
}
```

**`Theme`** / **`ShellConfiguration`** — top-level shell theming (title, icon, available themes list).

---

## 5. Creator API (Fluent Builder) Layer

The creator API builds valid `FlexRuntimeConfiguration` objects without hand-writing JSON. All creators follow a **builder pattern** with a terminal `.toConfig()` method.

### 5.1 FlexApplicationConfigCreator

```typescript
const appConfig = new FlexApplicationConfigCreator('/my-route');

const homeScreen = appConfig.createScreen('home', 'HomeScreen');
appConfig.setScreenToBootstrap(homeScreen);
appConfig.addControlDef(MyControlDefinitionModule);
appConfig.setShellConfig('My App', 'home');
appConfig.addTheme('dark', 'Dark Mode', { cssVariables: [...] });

const runtimeConfig: FlexRuntimeConfiguration = appConfig.toConfig(HyLogLevel.Warn);
```

Throws if `setScreenToBootstrap` was not called before `toConfig()`.

---

### 5.2 FlexScreenConfigCreator

```typescript
const screen = appConfig.createScreen('search', 'SearchScreen');

const searchInput = screen.addControlConfiguration({
  selector: 'my-search-input',
  id: 'searchInput',
  configData: { placeholder: 'Enter keyword' }
});

const opList = screen.addOperationList({ id: 'onSearch', systemName: 'OnSearch' });
screen.addHook({ event: LifecycleEvent.ScreenInit, operationListId: 'init-ops' });
screen.setHtmlTemplate(`<my-search-input flex-control-id="searchInput"></my-search-input>`);
```

Throws if `setHtmlTemplate` was not called before `toConfig()`.

---

### 5.3 ControlConfigCreator<T>

Generic over the control's `@Input()` interface `T`. Provides type-safe `addInputBinding()`, `addInboundEvent()`, `addOutboundEvent()`.

---

### 5.4 OperationListConfigCreator + OperationConfigCreator

```typescript
const opList = screen.addOperationList({ id: 'fetchData' });

opList.addOperation({
  operationType: 'ExecuteHttpRequest',
  data: { ... },
  storeResultAs: storeGloballyAs('searchResults')
});

opList.addEventMapping({
  controlId: 'searchInput',
  outboundEventName: 'searchTriggered'
});
```

---

### 5.5 HooksConfigCreator

```typescript
screen.addHook({
  event: LifecycleEvent.ScreenInit,
  operationListId: 'init-ops'
});
```

---

### 5.6 EventMappingConfigCreator

Wires `@Output()` events from controls directly to operation list invocations.

---

### 5.7 Config Creator Helpers

Standalone functions used inside operation config `data` objects:

```typescript
// Bind to a dynamic runtime property (read from store at execution time)
property('$global.userId')

// Bind to a literal value (baked into config)
constant('OnBase Web Server')

// Write operation output to the local (operation-list-scoped) store
storeLocallyAs('tempResult')

// Write operation output to the global (screen-scoped) store
storeGloballyAs('userId')
```

`$store.<name>` accesses local store values; `$global.<name>` accesses global store values.

---

## 6. Full FlexRuntimeConfiguration Shape

```
  FlexRuntimeConfiguration
  ├── config: ApplicationConfiguration
  │   ├── bootstrapScreen: string
  │   ├── route: string
  │   ├── controlDefinitions: ControlDefinition[]
  │   │     └── selector, properties[], inboundEvents[], outboundEvents[], methods[]
  │   └── screens: ScreenConfiguration[]
  │         ├── id, systemName, htmlContentTemplate
  │         ├── hooks: LifecycleHook[]
  │         │     └── event (LifecycleEvent), operationListId
  │         ├── operationLists: OperationListConfiguration[]
  │         │     ├── id, systemName, shouldCancelPreviousInvocationsOfList
  │         │     ├── operations: OperationConfiguration[]
  │         │     │     └── operationType, data (OperationConfigData)
  │         │     └── eventMappings: OperationEventMapping[]
  │         ├── controlConfigurations: ControlConfiguration[]
  │         │     ├── id, selector, styles
  │         │     ├── data: ControlConfigurationData[] (property bindings)
  │         │     ├── formData: ControlConfigurationFormData[]
  │         │     ├── inboundEvents: (outbound event → operation list)
  │         │     └── outboundEvents: (control → operation list trigger)
  │         └── routes: RouteConfiguration[]
  │               └── path, screenId, breakpoints: [{breakpoint, flexRouterOutletControlId}]
  ├── envData: EnvironmentVariableCollection
  ├── minimumLoggingLevel: HyLogLevel
  ├── shellConfig: ShellConfiguration (title, icon, themes[])
  ├── stateActions: FlexStateAction[]
  └── onBaseSettings: { contentIntelligenceConnector }
```

---

## 7. Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Pure data, zero runtime** | No Angular modules, services, or DI tokens |
| **Two layers** | Raw models (JSON-compatible) + fluent builders (TypeScript ergonomics) |
| **Type safety** | Generic `ControlConfigCreator<T>` prevents wrong binding names |
| **Creator helpers** | `property()`, `constant()`, `storeLocallyAs()`, `storeGloballyAs()` encode intent |
| **Fail-fast builders** | `toConfig()` throws descriptively if required fields are missing |
| **UUID defaults** | All `id` fields auto-generate UUIDs if not provided explicitly |

---

*Generated from codebase analysis — `develop` branch, March 2026.*
