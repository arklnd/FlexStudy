# `flex-host` — Architecture Deep Dive

> **Audience**: Engineers integrating or maintaining the top-level Flex embedding layer.  
> **Package**: `@hyland/flex-host`  
> **LOC**: 93 lines (TS src) · 48 spec · 1 HTML · 0 SCSS  
> **Usage score**: 5 import-lines across the repo · Tier 4 — Low Effort  
> **Companion document**: See [`libs/ARCHITECTURE.md`](../ARCHITECTURE.md) for the full cross-library picture.

---

## Table of Contents

1. [What Is `flex-host`?](#1-what-is-flex-host)
2. [The Big Picture](#2-the-big-picture)
3. [Single Exported Symbol](#3-single-exported-symbol)
4. [Component Responsibilities](#4-component-responsibilities)
5. [Inputs & Data Flow](#5-inputs--data-flow)
6. [Providers Wired by FlexHostComponent](#6-providers-wired-by-flexhostcomponent)
7. [Template Analysis](#7-template-analysis)
8. [Integration Pattern](#8-integration-pattern)
9. [Design Principles](#9-design-principles)

---

## 1. What Is `flex-host`?

`flex-host` is the **thin public-facing embedding layer** for the entire Flex runtime. It exports exactly one Angular component, `FlexHostComponent`, which is the recommended mount point for consumer applications that want to run a Flex application inside their host page.

Think of it as the **minimal surface area** an integrating app needs to touch. All the complex rendering, routing, and lifecycle machinery lives in `flex-runtime`; `flex-host` merely bridges configuration from the outer world into the runtime.

---

## 2. The Big Picture

```
  Consumer Angular App (onbase-apps-host, client-app, etc.)
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │   <flex-host [appConfig]="myConfig"></flex-host>            │
  │          │                                                  │
  └──────────┼──────────────────────────────────────────────────┘
             │   @Input() appConfig: FlexRuntimeConfiguration
             ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  FlexHostComponent   (flex-host)                             │
  │                                                              │
  │  • Accepts FlexRuntimeConfiguration from parent              │
  │  • Calls IFlexRuntimeConfigService.setConfig()               │
  │  • Configures PathLocationStrategy                           │
  │  • Installs CanScreenDeactivateGuard                         │
  │  • Renders <flex-app-viewer> when config is present          │
  └──────────────────────────────────────────────────────────────┘
             │
             ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  FlexAppViewerComponent   (flex-runtime)                     │
  │  • Screens, routing, operations, forms, event bus            │
  └──────────────────────────────────────────────────────────────┘
```

---

## 3. Single Exported Symbol

```
  flex-host/src/index.ts
  └── FlexHostComponent
```

That is the **entire public surface** of this library. There is no module, no service, no token exported.

---

## 4. Component Responsibilities

| # | Responsibility | How |
|---|----------------|-----|
| 1 | Accept configuration from host app | `@Input() appConfig: FlexRuntimeConfiguration` |
| 2 | Push config into runtime | `IFlexRuntimeConfigService.setConfig()` (async) |
| 3 | Guard dirty screens from navigation loss | Provide `CanScreenDeactivateGuard` in `FLEX_ROUTE_GUARDS_TOKEN` |
| 4 | Use real URL routing | Provide `PathLocationStrategy` (overrides any HashLocationStrategy) |
| 5 | Gate the viewer on config availability | Template renders `<flex-app-viewer>` only when config is truthy |

---

## 5. Inputs & Data Flow

```
  Host App sets [appConfig]
       │
       ▼
  FlexHostComponent setter
  │  detects change (prev !== next)
  │  strips null → undefined
  └──► IFlexRuntimeConfigService.setConfig(config)  [async, awaited]
             │
             ▼
       flex-runtime picks up new config
       ─► router re-initializes
       ─► screens rendered
```

The setter is the only external interaction surface. There are no `@Output()` events. The component is intentionally **write-only from the host's perspective**.

---

## 6. Providers Wired by FlexHostComponent

`FlexHostComponent` is a standalone component and its `providers` array configures the entire Flex DI sub-tree when an instance is created:

```
  providers: [
    {
      provide:  FLEX_ROUTE_GUARDS_TOKEN,
      useValue: {
        canActivate:   [],                    // no extra activation guards
        canDeactivate: [CanScreenDeactivateGuard]  // from flex-runtime
      }
    },
    {
      provide:  LocationStrategy,
      useClass: PathLocationStrategy          // from @angular/common
    }
  ]
```

### Why `PathLocationStrategy`?

Some integrating apps use `HashLocationStrategy` globally. Flex uses real HTML5 push-state URLs internally. By providing `PathLocationStrategy` at the component level, the Flex router sub-tree always gets real URLs even if the parent app prefers hash routing.

### Why `CanScreenDeactivateGuard`?

Flex screens can have dirty state (unsaved form data). The guard intercepts Angular Router navigation events from within the `flex-runtime` router outlet and prompts the user before allowing navigation away.

---

## 7. Template Analysis

```html
<!-- flex-host.component.html -->
<flex-app-viewer *ngIf="!!appConfig"></flex-app-viewer>
```

The template is **one line**. The `*ngIf` ensures no rendering (and no router initialization) happens until `appConfig` is non-null. This prevents race conditions during app startup where config may arrive asynchronously.

---

## 8. Integration Pattern

A consumer app mounts the Flex runtime as follows:

```typescript
// In the host app's component
@Component({
  imports: [FlexHostComponent],
  template: `<flex-host [appConfig]="config$ | async"></flex-host>`
})
export class HostAppComponent {
  config$ = this.configService.load();
}
```

No additional module imports, no service registrations — the component self-provisions everything it needs through its own `providers` array.

---

## 9. Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Minimal surface** | Single component, no exported services or tokens |
| **Self-contained DI** | All required providers declared in the component's own `providers` |
| **Null-safe config** | Setter converts `null` to `undefined`; template guards on truthiness |
| **Async-safe** | Config setter is async; awaits `setConfig` before runtime acts |
| **No vendor lock-in** | The `@Input()` contract is just a plain TypeScript config object |

---

*Generated from codebase analysis — `develop` branch, March 2026.*
