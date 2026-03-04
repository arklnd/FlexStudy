# Flex Runtime — Architecture Deep Dive

> **Audience**: Engineers new to this codebase.  
> **Purpose**: Explain how `flex-runtime` orchestrates the entire Flex application stack.

---

## Table of Contents

1. [The Big Picture](#1-the-big-picture)
2. [Library Responsibilities at a Glance](#2-library-responsibilities-at-a-glance)
3. [Core Responsibility of `flex-runtime`](#3-core-responsibility-of-flex-runtime)
4. [Key Architectural Components](#4-key-architectural-components)
   - [FlexModule — The DI Hub](#a-flexmodule--the-dependency-injection-hub)
   - [FlexAppViewerComponent — The Rendering Engine](#b-flexappviewercomponent--the-rendering-engine)
5. [The Full Lifecycle (Step-by-Step)](#5-the-full-lifecycle-step-by-step)
6. [Navigation Guard: `DoesScreenContainDirtyFormResolver`](#6-navigation-guard-doesscreencontaindirtyformresolver)
7. [State Management Map](#7-state-management-map)
8. [Design Principles](#8-design-principles)

---

## 1. The Big Picture

```
 ┌─────────────────────────────────────────────────────────────────────┐
 │                        HOST APPLICATION                             │
 │          (client-app / dev-app / onbase-apps-host)                  │
 │                                                                     │
 │   imports FlexModule.forRoot(config)                                │
 │   places  <flex-app-viewer> in its template                         │
 └────────────────────────────┬────────────────────────────────────────┘
                              │  bootstraps
                              ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │                        flex-runtime  ◄── YOU ARE HERE               │
 │                                                                     │
 │  ┌──────────────┐   ┌──────────────────────┐   ┌────────────────┐  │
 │  │  FlexModule  │   │ FlexAppViewerComponent│   │  Nav Resolvers │  │
 │  │  (DI Hub)    │   │ (Rendering Engine)    │   │  (Guards)      │  │
 │  └──────┬───────┘   └──────────┬───────────┘   └───────┬────────┘  │
 └─────────┼────────────────────┼──────────────────────────┼──────────┘
           │                    │                           │
     ┌─────▼──────┐      ┌──────▼──────┐          ┌────────▼────────┐
     │flex-config │      │ flex-router │          │  flex-forms     │
     │flex-ops    │      │(FlexScreen  │          │(FormToScreen    │
     │flex-types  │      │ Component)  │          │ Registry)       │
     └────────────┘      └─────────────┘          └─────────────────┘
```

**In one sentence**: `flex-runtime` is the **coordinator**. It does not know your business logic — it just starts everything, wires everything together, and cleans everything up.

---

## 2. Library Responsibilities at a Glance

```
 ┌──────────────────┬────────────────────────────────────────────────────┐
 │ Library          │ What it owns                                        │
 ├──────────────────┼────────────────────────────────────────────────────┤
 │ flex-runtime     │ Bootstrap, lifecycle, DI registration, nav guards  │
 │ flex-host        │ Shell / micro-frontend container                    │
 │ flex-router      │ Screen navigation, FlexScreenComponent              │
 │ flex-config      │ ConfigurationService, remote config fetching        │
 │ flex-forms       │ Dynamic form generation & validation                │
 │ flex-operations  │ NgRx stores, effects, business-level actions        │
 │ flex-shared      │ Shared utilities, interceptors, helpers             │
 │ flex-types       │ Pure TypeScript interfaces & type definitions       │
 └──────────────────┴────────────────────────────────────────────────────┘
```

---

## 3. Core Responsibility of `flex-runtime`

`flex-runtime` **does not contain business logic**. Think of it as the "city electricity grid":  
- It does not generate the power (that's `flex-operations`).  
- It does not use the power (that's your host app / screens).  
- It **wires up, distributes, and shuts down** the power safely.

```
  WHAT flex-runtime DOES:
  ═══════════════════════
  [1] Initialise NgRx state stores for the whole app
  [2] Bind interface tokens → concrete implementations (DI)
  [3] Expose <flex-app-viewer> as the single mount point
  [4] Fetch remote config and hand screen info to flex-router
  [5] Clean up all state and routes when the viewer is destroyed
```

---

## 4. Key Architectural Components

### A. `FlexModule` — The Dependency Injection Hub

```
  FlexModule.forRoot(retrievalConfig)
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │  NgRx Store Slices registered:                           │
  │  ┌────────────────────────┐                              │
  │  │ ConfigurationStore     │  ← screen / app config       │
  │  │ ComponentTrackingStore │  ← tracks live components    │
  │  │ FlexRuntimeConfigStore │  ← runtime-level settings    │
  │  │ EnvironmentVariablesStore← env-specific values        │
  │  │ FlexPropertyBindingStore ← dynamic data bindings      │
  │  └────────────────────────┘                              │
  │                                                          │
  │  DI Token bindings:                                      │
  │  ┌─────────────────────────────────────────────────┐     │
  │  │ FLEX_CONFIG_SERVICE  →  ConfigurationService    │     │
  │  │ AUTH_SERVICE         →  (overridable by host)   │     │
  │  │ ENV_VARS_SERVICE     →  (overridable by host)   │     │
  │  │ SCREEN_SERVICE       →  (overridable by host)   │     │
  │  └─────────────────────────────────────────────────┘     │
  │                                                          │
  │  ⚠  Host apps can OVERRIDE any token in forRoot()        │
  │     so their custom auth/env logic takes effect.         │
  └──────────────────────────────────────────────────────────┘
```

**Why tokens?** If you use `new MyService()` everywhere, you can never swap it for a mock in tests.  
Tokens let you say "whatever is registered for `AUTH_SERVICE`, inject that" — the caller never hard-codes the class.

---

### B. `FlexAppViewerComponent` — The Rendering Engine

```
  <flex-app-viewer>   (selector: flex-app-viewer)
  ┌──────────────────────────────────────────────┐
  │                                              │
  │  ViewEncapsulation.None                      │
  │  (CSS is not scoped — needed for dynamic     │
  │   controls rendered at runtime)              │
  │                                              │
  │  Listens to:  configLoaded$  ──────────┐     │
  │                                        │     │
  │  On config loaded:                     ▼     │
  │  loadContentFromApplicationAsync(config)     │
  │    │                                         │
  │    ├─ reads config.bootstrapScreen           │
  │    └─ calls screenControl.setScreenInfo(...) │
  │           │                                  │
  │           ▼                                  │
  │      flex-router (FlexScreenComponent)       │
  │      injects UI controls into this.elRef     │
  └──────────────────────────────────────────────┘
```

---

## 5. The Full Lifecycle (Step-by-Step)

```
  PHASE 1 — INIT
  ══════════════
  Host App
    │
    ├─ imports FlexModule.forRoot(config)
    │         │
    │         ├─ registers NgRx stores
    │         ├─ binds DI tokens
    │         └─ starts Effects (async side effects / API calls)
    │
    └─ renders <flex-app-viewer> in HTML template


  PHASE 2 — CONFIG FETCH
  ══════════════════════
  FlexAppViewerComponent (ngOnInit)
    │
    └─ subscribes to configLoaded$
              │
              │  [HTTP request fires via ConfigurationService]
              │
              ▼
         Config arrives from backend
         {
           bootstrapScreen: { id, type, ... },
           endpoints: { ... },
           dataBindings: { ... }
         }


  PHASE 3 — SCREEN BOOTSTRAP
  ══════════════════════════
  FlexAppViewerComponent
    │
    ├─ reads config.bootstrapScreen
    │
    └─ screenControl.setScreenInfo(bootstrapScreen)
              │
              ▼
         flex-router
         FlexScreenComponent
              │
              ├─ parses screen definition
              ├─ dynamically instantiates controls
              │    ├─ flex-forms  (if form controls)
              │    ├─ data grids  (if list controls)
              │    └─ custom widgets (if any)
              │
              └─ injects them into DOM (elRef)


  PHASE 4 — USER INTERACTION
  ══════════════════════════
  User navigates between screens
    │
    └─ DoesScreenContainDirtyFormResolver runs
              │
              ├─ YES dirty  →  block navigation / prompt user
              └─ NO dirty   →  allow navigation


  PHASE 5 — TEARDOWN
  ══════════════════
  <flex-app-viewer> destroyed (ngOnDestroy)
    │
    ├─ routerConfigService.deregisterApplication()
    │    └─ removes this app's routes from Angular Router
    │
    └─ dataStoreService.dispatch(clearOperationStore())
         └─ wipes NgRx state so next app loads clean
              (prevents data leakage between micro-frontends)
```

---

## 6. Navigation Guard: `DoesScreenContainDirtyFormResolver`

This resolver lives in `libs/flex-runtime/src/lib/allow-navigation-resolvers/`.

```
  User clicks "Navigate Away"
         │
         ▼
  DoesScreenContainDirtyFormResolver.canNavigateAwayFromScreen(screen)
         │
         └─ asks FormToScreenRegistryService:
                 "Are ANY forms on screen [screenInstanceId] dirty?"
                        │
               ┌────────┴────────┐
               │                 │
              YES               NO
               │                 │
               ▼                 ▼
          return false       return true
          (BLOCK nav)        (ALLOW nav)
```

**Key insight**: `flex-runtime` does not know what forms exist. It delegates to `FormToScreenRegistryService` (from `flex-forms`), which maintains a live registry of all form instances per `screenInstanceId`. This keeps `flex-runtime` decoupled from form implementation details.

---

## 7. State Management Map

```
  NgRx STORE (managed by flex-runtime via FlexModule)
  ┌─────────────────────────────────────────────────────┐
  │                                                     │
  │  ConfigurationStore        OperationStore           │
  │  ┌──────────────────┐     ┌──────────────────────┐  │
  │  │ app config       │     │ HTTP operation state  │  │
  │  │ screen defs      │     │ loading / error flags │  │
  │  │ endpoints        │     │ CLEARED on teardown ◄─┼──┼── prevents leakage
  │  └──────────────────┘     └──────────────────────┘  │
  │                                                     │
  │  ComponentTrackingStore    FlexPropertyBindingStore  │
  │  ┌──────────────────┐     ┌──────────────────────┐  │
  │  │ which components │     │ dynamic data bindings │  │
  │  │ are mounted      │     │ between screen fields │  │
  │  └──────────────────┘     └──────────────────────┘  │
  │                                                     │
  │  EnvironmentVariablesStore  FlexRuntimeConfigStore  │
  │  ┌──────────────────┐      ┌──────────────────────┐ │
  │  │ env-specific     │      │ runtime-level feature │ │
  │  │ configuration    │      │ flags & settings      │ │
  │  └──────────────────┘      └──────────────────────┘ │
  └─────────────────────────────────────────────────────┘
```

---

## 8. Design Principles

```
  ┌────────────────────────────────────────────────────────────────┐
  │  PRINCIPLE                   HOW IT APPEARS IN CODE            │
  ├────────────────────────────────────────────────────────────────┤
  │  Configuration-Driven UI   │ Screen shape comes from backend   │
  │                            │ config, not hard-coded templates  │
  ├────────────────────────────────────────────────────────────────┤
  │  Inversion of Control      │ Injection Tokens everywhere;      │
  │                            │ hosts provide own implementations │
  ├────────────────────────────────────────────────────────────────┤
  │  State-Driven (NgRx)       │ All side effects go through store │
  │                            │ actions; no direct HTTP in UI     │
  ├────────────────────────────────────────────────────────────────┤
  │  Clean-Up as First Class   │ ngOnDestroy explicitly clears     │
  │                            │ routes AND state — mic-frontend   │
  │                            │ isolation guaranteed              │
  ├────────────────────────────────────────────────────────────────┤
  │  Delegated Responsibility  │ flex-runtime does NOT implement   │
  │                            │ forms, routing rules, or HTTP.    │
  │                            │ It only coordinates the libraries │
  │                            │ that do.                          │
  └────────────────────────────────────────────────────────────────┘
```

---

*Generated from codebase analysis — `develop` branch, March 2026.*
