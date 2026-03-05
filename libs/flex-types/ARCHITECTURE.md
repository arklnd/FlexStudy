# `flex-types` — Architecture Deep Dive

> **Audience**: Engineers new to this codebase.  
> **Package**: `@hyland/flex-types`  
> **LOC**: 91 lines (TS src) · 0 specs · 0 HTML · 0 SCSS  
> **Usage score**: 27 import-lines across the repo · Tier 3 — Domain-Scoped

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [What It Owns](#2-what-it-owns)
3. [Every Exported Type — Annotated](#3-every-exported-type--annotated)
4. [Zero Runtime Weight](#4-zero-runtime-weight)
5. [Who Imports This Library](#5-who-imports-this-library)
6. [Dependency Graph](#6-dependency-graph)
7. [Re-engineering Notes](#7-re-engineering-notes)

---

## 1. Purpose

`flex-types` is a **pure TypeScript utility library**. It contains only type-level constructs — no Angular, no services, no runtime logic. Every export is erased completely at compile time (TypeScript `type` aliases or interfaces).

Its sole job: give the rest of the codebase a shared vocabulary of reusable generic types so engineers don't write the same boilerplate transforms in every file.

```
  flex-types
  ┌─────────────────────────────────────────────────────────┐
  │  Pure TypeScript types — zero runtime output            │
  │  No Angular dependency                                  │
  │  No side-effects                                        │
  │  Import cost: exactly 0 bytes in the final bundle       │
  └─────────────────────────────────────────────────────────┘
```

---

## 2. What It Owns

```
  flex-types/src/lib/
  ┌──────────────────────────────────────────────────────────────────┐
  │  extract-emitted-type.type.ts   ExtractEmittedValue<T>           │
  │  omit-function-props.type.ts    OmitFunctionProperties<T>        │
  │  omit-props-of-type.type.ts     OmitPropertiesOfType<T, V>       │
  │  pick-observable-props.type.ts  PickObservablePropertyNames<T>   │
  │  pick-props-of-type.type.ts     PickPropertiesOfType<T, V>       │
  │  typed-simple-changes.type.ts   TypedSimpleChanges<T>            │
  │  union.type.ts                  UnionType<T>                     │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 3. Every Exported Type — Annotated

### `TypedSimpleChanges<T>`
A typed replacement for Angular's `SimpleChanges`. Avoids casting in `ngOnChanges`.

```typescript
// Without flex-types — unsafe cast required
ngOnChanges(changes: SimpleChanges) {
  const val = changes['myInput']?.currentValue as string;
}

// With flex-types — fully type-safe
ngOnChanges(changes: TypedSimpleChanges<MyComponent>) {
  const val = changes.myInput?.currentValue; // inferred as string
}
```

**Used in**: Every component that has `@Input()` properties and implements `ngOnChanges`.

---

### `ExtractEmittedValue<T>`
Extracts the emitted `T` from an `EventEmitter<T>` or `Observable<T>`.

```typescript
type ButtonClick = ExtractEmittedValue<ButtonComponent['clicked']>;
// ButtonClick = MouseEvent
```

**Used in**: `source-events.ts` in `onbase-apps-host` to type event payloads without importing component classes.

---

### `OmitFunctionProperties<T>`
Strips all method/function members from an interface, leaving only data properties.

```typescript
interface MyService { id: number; load(): void; data$: Observable<string>; }
type DataOnly = OmitFunctionProperties<MyService>;
// DataOnly = { id: number; data$: Observable<string> }
```

**Used in**: `flex-config`'s config creator builders and `onbase-web-server`'s dialog data interfaces.

---

### `PickObservablePropertyNames<T>`
Returns a union of property names whose values are of type `Observable<any>`.

```typescript
type ObsKeys = PickObservablePropertyNames<{ id: number; data$: Observable<string>; flag: boolean }>;
// ObsKeys = 'data$'
```

**Used in**: `flex-config` control creators, `onbase-apps-host` source registry.

---

### `OmitPropertiesOfType<T, V>`
Removes all properties whose value type is `V`.

```typescript
type NoFunctions = OmitPropertiesOfType<MyClass, Function>;
```

---

### `PickPropertiesOfType<T, V>`
Keeps only properties whose value type is `V`.

---

### `UnionType<T>`
Helper for creating discriminated union types from object definitions.

---

## 4. Zero Runtime Weight

Because every export is a TypeScript `type` alias:

```
  TypeScript compiler erase all types at compilation.
  flex-types adds ZERO bytes to the production JavaScript bundle.
  There is no Angular module, no injectable service, nothing to bootstrap.
```

This means `flex-types` is unique: it has **27 dependents** but **zero production impact**. Removing it would only require search-and-replace of the types inline — no logic migration needed.

---

## 5. Who Imports This Library

```
  ┌──────────────────────────────────┬──────────────────────────────────┐
  │ Importer                         │ What they use                    │
  ├──────────────────────────────────┼──────────────────────────────────┤
  │ flex-config                      │ PickObservablePropertyNames      │
  │                                  │ OmitFunctionProperties           │
  ├──────────────────────────────────┼──────────────────────────────────┤
  │ shared-components                │ TypedSimpleChanges               │
  │                                  │ UnionType                        │
  ├──────────────────────────────────┼──────────────────────────────────┤
  │ onbase-web-server                │ TypedSimpleChanges               │
  │                                  │ OmitFunctionProperties           │
  │                                  │ PickObservablePropertyNames      │
  ├──────────────────────────────────┼──────────────────────────────────┤
  │ onbase-apps-shell                │ TypedSimpleChanges               │
  ├──────────────────────────────────┼──────────────────────────────────┤
  │ onbase-apps-host (app)           │ TypedSimpleChanges               │
  │                                  │ ExtractEmittedValue              │
  │                                  │ OmitFunctionProperties           │
  │                                  │ PickObservablePropertyNames      │
  ├──────────────────────────────────┼──────────────────────────────────┤
  │ client-app (app)                 │ OmitFunctionProperties           │
  └──────────────────────────────────┴──────────────────────────────────┘
```

---

## 6. Dependency Graph

```
  flex-types
      │
      │  (none — zero inbound or outbound library dependencies)
      │
  Imported by:
      ├── flex-config
      ├── flex-shared
      ├── shared-components
      ├── onbase-web-server
      ├── onbase-apps-shell
      └── apps/onbase-apps-host, apps/client-app
```

`flex-types` is the **only library in the entire stack with zero dependencies**. It sits at the absolute bottom of the dependency tree.

---

## 7. Re-engineering Notes

| Question | Answer |
|----------|--------|
| Can it be removed? | Yes — replace usages inline, ~27 files to update |
| Runtime risk | Zero — types are compile-time only |
| Migration effort | Copy 7 type definitions into a shared `types/` folder inside each consuming library |
| Test impact | Zero — no runtime code to break |
| Bundle impact | Zero — already contributes 0 bytes |

---

*Generated from codebase analysis — `develop` branch, March 2026.*
