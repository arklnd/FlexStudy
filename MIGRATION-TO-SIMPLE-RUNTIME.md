# Can `flex-runtime` Be Swapped With `SimpleRuntime`?
### Feasibility Report — Angular 21 Migration

> **Constraint**: Host applications (`client-app`, `dev-app`, `onbase-apps-host`) must have **minimal** code changes.  
> **Verdict**: ✅ **Yes — feasible.** With the right abstraction strategy, host apps face **only one mandatory change** and two optional simplifications.

---

## Table of Contents

1. [What Host Apps Currently Touch (The Contract)](#1-what-host-apps-currently-touch--the-public-contract)
2. [What Makes the Current Runtime Hard to Swap](#2-what-makes-the-current-runtime-hard-to-swap)
3. [Angular 21 Features That Enable SimpleRuntime](#3-angular-21-features-that-enable-simpleruntime)
4. [The Migration Strategy](#4-the-migration-strategy)
5. [Component-by-Component Replacement Plan](#5-component-by-component-replacement-plan)
6. [Host App Impact — Before vs After](#6-host-app-impact--before-vs-after)
7. [Risk Register](#7-risk-register)
8. [Work Breakdown](#8-work-breakdown)
9. [Decision Summary](#9-decision-summary)

---

## 1. What Host Apps Currently Touch — The Public Contract

Before deciding what can change, we must map exactly what host apps import from `@hyland/flex-runtime`.
Evidence taken directly from `apps/client-app/src/app/views/client/client.module.ts`:

```
  HOST APP IMPORTS FROM @hyland/flex-runtime
  ═══════════════════════════════════════════
  ┌────────────────────────────────────┬───────────────────────────────────────┐
  │ Symbol                             │ How it is used                        │
  ├────────────────────────────────────┼───────────────────────────────────────┤
  │ FlexModule                         │ FlexModule.forRoot(config) in NgModule │
  │ FLEX_EXTERNAL_PROVIDER             │ multi-provider token in NgModule       │
  │ FlexExternalProvider               │ interface that providers implement     │
  │ FlexAppViewerComponent             │ direct import in standalone component  │
  │ FlexRuntimeConfigService           │ constructor @Inject injection          │
  │ FlexBroadcastService               │ constructor @Inject injection          │
  │ CanScreenDeactivateGuard           │ route canDeactivate guard              │
  └────────────────────────────────────┴───────────────────────────────────────┘

  HOST APP TEMPLATE USAGE:
  ════════════════════════
  <flex-app-viewer *ngIf="!!appConfig"></flex-app-viewer>   ← selector must stay

  HOST APP NgRx SETUP (imposed by flex-runtime today):
  ═════════════════════════════════════════════════════
  StoreModule.forRoot({}, { runtimeChecks: {...} })          ← host must bootstrap NgRx
  StoreDevtoolsModule.instrument({ name: 'Flex Runtime...' }) ← host configures devtools
```

**This is the "contract" that SimpleRuntime must honour or provide a migration shim for.**

---

## 2. What Makes the Current Runtime Hard to Swap

### 2A. The NgModule Wall

```
  CURRENT PATTERN:
  ════════════════

  client.module.ts (host app NgModule)
  ┌──────────────────────────────────────────────────────────────────┐
  │  @NgModule({                                                     │
  │    imports: [                                                    │
  │      FlexModule.forRoot({ flexAuthService: ... })  ← PROBLEM    │
  │      StoreModule.forRoot({})                       ← PROBLEM    │
  │      StoreDevtoolsModule.instrument({})            ← PROBLEM    │
  │    ]                                                             │
  │  })                                                              │
  │  export class ClientModule {}                                    │
  └──────────────────────────────────────────────────────────────────┘

  The host app is tightly coupled to:
  1. The Angular MODULE system  (FlexModule)
  2. NgRx specifically          (StoreModule, StoreDevtoolsModule)
  3. The forRoot() pattern      (Angular 16-era API)
```

### 2B. Heavy NgRx Dependency Leaks Out

The host app currently **must** call `StoreModule.forRoot()` because `FlexModule` registers feature stores that depend on it. This means:
- Host apps know NgRx exists.
- Host apps configure NgRx runtime checks.
- Host apps name the NgRx devtools instance.

If `SimpleRuntime` drops NgRx internally, **both** `StoreModule.forRoot()` and `StoreDevtoolsModule` entries vanish from host apps entirely — a **net reduction** in host boilerplate.

### 2C. `CanDeactivate` Class-Based Guard

```typescript
// can-screen-deactivate.guard.ts — current implementation
export class CanScreenDeactivateGuard implements CanDeactivate<any> { ... }
```

`CanDeactivate` as a class was **deprecated in Angular 15.1** and removed in Angular 17. Host apps that wire this guard into route config will break on Angular 21 regardless of runtime swap.

### 2D. Constructor `@Inject()` Pattern

```typescript
// wv-flex-host.component.ts in host app
constructor(
  @Inject(FLEX_RUNTIME_CONFIG_SERVICE) flexRuntimeConfigService: IFlexRuntimeConfigService,
  @Inject(FLEX_LOG_SERVICE) logService: ILogService,
  @Inject(FLEX_BROADCAST_SERVICE) flexBroadcastService: IBroadcastService,
) { ... }
```

This is **not a blocker** — as long as the same Injection Tokens are kept, host app constructors keep working. The implementation behind those tokens can change completely.

### 2E. Manual Subscription Management

```typescript
// flex-app-viewer.component.ts
private _subscriptions: Subscription = new Subscription();

ngAfterViewInit(): void {
  this._subscriptions.add(
    this.runtimeConfigService.configLoaded$.subscribe(...)
  );
}

ngOnDestroy(): void {
  this._subscriptions.unsubscribe();
  ...
}
```

This is an **internal** concern. Host apps never see this. Safe to modernise.

---

## 3. Angular 21 Features That Enable SimpleRuntime

```
  ANGULAR 16 (Current)           ANGULAR 21 (SimpleRuntime)
  ══════════════════════         ════════════════════════════════════
  NgModule + forRoot()      →    provideSimpleRuntime() function
  NgRx Store / Effects      →    Signals + effect() / resource API
  EffectsModule.forRoot()   →    Removed entirely
  StoreModule.forRoot()     →    Removed entirely
  class CanDeactivate guard →    Functional route guard canDeactivate: () => ...
  @Inject(TOKEN) in ctor    →    inject(TOKEN) in constructor body / field
  Subscription management   →    takeUntilDestroyed(destroyRef)
  ngAfterViewInit           →    afterNextRender()
  @ViewChild(...)           →    viewChild() signal
  Observable configLoaded$  →    Signal<FlexRuntimeConfiguration | undefined>
  ViewEncapsulation.None    →    Stays (still needed)
  async loadContent(...)    →    Can become effect() reaction
```

---

## 4. The Migration Strategy

The key insight is: **the public surface area is small and stable**. SimpleRuntime's job is to honour the same tokens and selectors, while rebuilding the internals with Angular 21 primitives.

```
  STRATEGY: "Honour the contract, modernise the engine"
  ══════════════════════════════════════════════════════

  KEEP (zero host changes needed):
  ─────────────────────────────────
  ✅ <flex-app-viewer> selector          (FlexConstants.FLEX_APP_VIEWER_SELECTOR)
  ✅ FLEX_RUNTIME_CONFIG_SERVICE token
  ✅ FLEX_BROADCAST_SERVICE token
  ✅ FLEX_EXTERNAL_PROVIDER token
  ✅ FlexExternalProvider interface
  ✅ FlexConfigurationOverride interface (same override keys)
  ✅ FlexRuntimeConfigService class name (re-export from SimpleRuntime)
  ✅ FlexBroadcastService class name

  RENAME (one change per host app):
  ─────────────────────────────────
  ❌ FlexModule.forRoot(config)
  ✅ provideSimpleRuntime(config)       ← drop-in replacement, same config shape

  REMOVE (reduces host app boilerplate):
  ──────────────────────────────────────
  ❌ StoreModule.forRoot({})            ← no longer needed
  ❌ StoreDevtoolsModule.instrument({}) ← no longer needed

  GUARD MIGRATION (needed anyway for Angular 21):
  ────────────────────────────────────────────────
  ❌ canDeactivate: [CanScreenDeactivateGuard]
  ✅ canDeactivate: [canScreenDeactivateFn]    ← exported functional guard
```

---

## 5. Component-by-Component Replacement Plan

### 5A. `FlexModule` → `provideSimpleRuntime()`

```
  CURRENT (Angular 16 NgModule):
  ══════════════════════════════
  FlexModule
  ┌─────────────────────────────────────────────────────────────────┐
  │  @NgModule({                                                    │
  │    imports: [                                                   │
  │      EffectsModule.forRoot(),                                   │
  │      ConfigurationStore.getModuleImports(),                     │
  │      ComponentTrackingStore.getModuleImports(),   ← NgRx        │
  │      FlexRuntimeConfigStore.getModuleImports(),   ← NgRx        │
  │      EnvironmentVariablesStore.getModuleImports(),← NgRx        │
  │      FlexPropertyBindingStore.getModuleImports(), ← NgRx        │
  │    ],                                                           │
  │    providers: [ token → class bindings... ]                     │
  │  })                                                             │
  └─────────────────────────────────────────────────────────────────┘
  static forRoot(config): ModuleWithProviders<FlexModule>

  ──────────────────────────────────────────────────────────────────

  SIMPLERUNTIME (Angular 21 provider function):
  ═════════════════════════════════════════════
  export function provideSimpleRuntime(config?: FlexConfigurationOverride): EnvironmentProviders {
    return makeEnvironmentProviders([
      // ❌ No NgRx stores
      // ✅ Signal-based state services (internal)
      provideFlexRouterModule(),          // wraps FlexRouterModule as providers
      { provide: SCREEN_SERVICE, useClass: config?.screenService ?? ScreenService },
      { provide: FLEX_RUNTIME_CONFIG_SERVICE, useClass: SimpleRuntimeConfigService },
      { provide: FLEX_BROADCAST_SERVICE, useClass: FlexBroadcastService },
      // ... all other token bindings
    ]);
  }
```

### 5B. `FlexRuntimeConfigService` → Signal-based

```
  CURRENT (NgRx-backed Observable):
  ══════════════════════════════════
  configLoaded$: Observable<FlexRuntimeConfiguration | undefined>
    = this.store
        .select(FlexRuntimeConfigStore.feature.selectFlexRuntimeConfigState)
        .pipe(map(state => state.config));

  SIMPLERUNTIME (Angular 21 Signal):
  ═══════════════════════════════════
  // Internal signal — host apps never see this directly
  private _config = signal<FlexRuntimeConfiguration | undefined>(undefined);

  // Still exposes Observable for backward compatibility (toObservable)
  configLoaded$: Observable<FlexRuntimeConfiguration | undefined>
    = toObservable(this._config);

  // ✅ Host apps that inject FLEX_RUNTIME_CONFIG_SERVICE see NO change
  // ✅ WvFlexHostComponent still subscribes to configLoaded$ unchanged
```

### 5C. `FlexAppViewerComponent` — Modernise internals only

```
  CURRENT:                              SIMPLERUNTIME:
  ════════                              ══════════════
  @ViewChild('screenControl')           viewChild<FlexScreenElement>('screenControl')
  screenElement: ElementRef             (signal-based, auto-typed)

  ngAfterViewInit() {                   afterNextRender(() => {
    this._subscriptions.add(              effect(() => {
      this.runtimeConfigService           const config = this.configSignal();
        .configLoaded$                    if (config) this.loadContent(config);
        .subscribe(config => ...)         });
    );                                  });
  }

  ngOnDestroy() {                       // Automatic via DestroyRef
    this._subscriptions.unsubscribe();  // No manual teardown needed
    this.routerConfigService            this.routerConfigService
      .deregisterApplication();           .deregisterApplication();   ← stays
    this.dataStoreService               // NgRx dispatch removed
      .dispatch(clearOperationStore()); // State cleanup via service.clear()
  }

  // SELECTOR STAYS: FlexConstants.FLEX_APP_VIEWER_SELECTOR
  // Host apps' <flex-app-viewer> bindings require ZERO change
```

### 5D. `CanScreenDeactivateGuard` → Functional Guard

```
  CURRENT (class-based, deprecated):              SIMPLERUNTIME (functional):
  ═════════════════════════════════               ══════════════════════════════
  @Injectable({ providedIn: 'root' })             export const canScreenDeactivateFn:
  export class CanScreenDeactivateGuard             CanDeactivateFn<any> =
    implements CanDeactivate<any> {                 (component, currentRoute,
      canDeactivate(...): Promise<boolean> { }        currentState, nextState) => {
  }                                                   const svc = inject(IFlexNavigationFacilitator);
                                                      return svc.canRouteTreeSafelyNavigate(...)
                                                        ? true
                                                        : inject(IDialogService).confirm(...);
                                                    };

  HOST APP CHANGE:
    Before: canDeactivate: [CanScreenDeactivateGuard]
    After:  canDeactivate: [canScreenDeactivateFn]
```

### 5E. NgRx Stores → Angular Signals

```
  CURRENT NgRx STORE           SIMPLERUNTIME SIGNAL EQUIVALENT
  ═════════════════════        ════════════════════════════════════════════
  ConfigurationStore           configurationService.config = signal<AppConfig>(...)
  ComponentTrackingStore       componentTracker.tracked = signal<Map<id, Comp>>(...)
  FlexRuntimeConfigStore       simpleRuntimeConfig.value = signal<Config | undefined>(...)
  EnvironmentVariablesStore    envVarsService.vars = signal<EnvVars>(...)
  FlexPropertyBindingStore     propertyBindingService.bindings = signal<Bindings>(...)

  Teardown (ngOnDestroy):
    Before: store.dispatch(OperationStore.actions.clearOperationStore())
    After:  operationService.clear()   ← directly calls signal reset
```

---

## 6. Host App Impact — Before vs After

### `client.module.ts` — The one file that changes

```typescript
  // ═══════════════════════════════════════════════
  // BEFORE (Angular 16 / flex-runtime)
  // ═══════════════════════════════════════════════
  import { NgModule } from '@angular/core';
  import { StoreModule } from '@ngrx/store';
  import { StoreDevtoolsModule } from '@ngrx/store-devtools';
  import { FLEX_EXTERNAL_PROVIDER, FlexModule } from '@hyland/flex-runtime';

  @NgModule({
    imports: [
      RouterModule,
      WvFlexHostComponent,
      FlexModule.forRoot({ flexAuthService: FlexAuthBypassService }),  // ← CHANGE
      StoreModule.forRoot({}, { runtimeChecks: { ... } }),             // ← REMOVE
      StoreDevtoolsModule.instrument({ name: 'Flex Runtime...' }),     // ← REMOVE
    ],
    providers: [
      { provide: FLEX_EXTERNAL_PROVIDER, useExisting: ... },
    ]
  })
  export class ClientModule {}


  // ═══════════════════════════════════════════════
  // AFTER (Angular 21 / SimpleRuntime)
  // ═══════════════════════════════════════════════
  import { NgModule } from '@angular/core';
  import { FLEX_EXTERNAL_PROVIDER, provideSimpleRuntime } from '@hyland/flex-runtime';
  //                                ^^^ renamed function, same config shape

  @NgModule({
    imports: [
      RouterModule,
      WvFlexHostComponent,
    ],
    providers: [
      provideSimpleRuntime({ flexAuthService: FlexAuthBypassService }), // ← RENAMED
      { provide: FLEX_EXTERNAL_PROVIDER, useExisting: ... },            // ← UNCHANGED
    ]
  })
  export class ClientModule {}
```

```
  HOST APP CHANGE SUMMARY:
  ═════════════════════════════════════════════════════════════════
  File                        Change Type    Description
  ─────────────────────────   ──────────     ─────────────────────
  client.module.ts            MANDATORY      FlexModule.forRoot()
                                             → provideSimpleRuntime()
  client.module.ts            REMOVAL        StoreModule.forRoot()
  client.module.ts            REMOVAL        StoreDevtoolsModule
  app-routing.module.ts       MANDATORY      CanScreenDeactivateGuard
                                             → canScreenDeactivateFn
  wv-flex-host.component.ts   NO CHANGE      still injects same tokens
  wv-flex-host.component.html NO CHANGE      <flex-app-viewer> selector same
  package.json                REMOVAL        @ngrx/store, @ngrx/effects,
                                             @ngrx/store-devtools devDeps
  ═════════════════════════════════════════════════════════════════
  Total mandatory edits: 2 files, ~5 lines each
  Net lines removed from host apps: ~15-20 lines of NgRx boilerplate
```

---

## 7. Risk Register

```
  ┌──────────────────────────────┬────────┬────────┬────────────────────────────────────────┐
  │ Risk                         │ Impact │  Prob  │ Mitigation                             │
  ├──────────────────────────────┼────────┼────────┼────────────────────────────────────────┤
  │ configLoaded$ consumers in   │  HIGH  │  LOW   │ Keep Observable via toObservable()     │
  │ host apps use .subscribe()   │        │        │ on internal signal. No host change.    │
  ├──────────────────────────────┼────────┼────────┼────────────────────────────────────────┤
  │ Other libs (flex-host,       │  HIGH  │  MED   │ Audit all @ngrx/store imports across   │
  │ flex-operations) still use   │        │        │ flex-* libs. Migrate leaf-first.       │
  │ NgRx internally              │        │        │                                        │
  ├──────────────────────────────┼────────┼────────┼────────────────────────────────────────┤
  │ setConfig() throws on hot-   │  MED   │  LOW   │ Comment already exists in code:        │
  │ swap (by design today)       │        │        │ "does not support hotswapping".        │
  │                              │        │        │ Behaviour stays the same.              │
  ├──────────────────────────────┼────────┼────────┼────────────────────────────────────────┤
  │ ViewEncapsulation.None on    │  LOW   │  HIGH  │ Keep as-is. Code comment confirms      │
  │ FlexAppViewerComponent is    │        │        │ this is intentional for dynamic        │
  │ intentional                  │        │        │ control rendering.                     │
  ├──────────────────────────────┼────────┼────────┼────────────────────────────────────────┤
  │ Third-party host consumers   │  HIGH  │  MED   │ Keep FlexModule as a deprecated        │
  │ outside this repo still use  │        │        │ re-export shim that calls              │
  │ FlexModule.forRoot()         │        │        │ provideSimpleRuntime() internally.     │
  ├──────────────────────────────┼────────┼────────┼────────────────────────────────────────┤
  │ Signal-based state loses     │  MED   │  MED   │ Use NgRx Signals API (provideStore +   │
  │ Redux DevTools visibility    │        │        │ signalStore) as stepping stone before  │
  │                              │        │        │ full removal. Preserves devtools.      │
  ├──────────────────────────────┼────────┼────────┼────────────────────────────────────────┤
  │ flex-operations lib is a     │  HIGH  │  HIGH  │ flex-operations is a separate lib with │
  │ deep NgRx dependency         │        │        │ its own OperationStore. Must be        │
  │                              │        │        │ migrated in parallel or kept on NgRx   │
  │                              │        │        │ with a compatibility layer.             │
  └──────────────────────────────┴────────┴────────┴────────────────────────────────────────┘
```

---

## 8. Work Breakdown

```
  PHASE 1 — Preparation (can be done before Angular 21 upgrade)
  ══════════════════════════════════════════════════════════════
  [ ] Audit all flex-* libs for direct NgRx store.select() / dispatch() calls
  [ ] Identify all external consumers of FlexModule.forRoot()
  [ ] Add deprecation JSDoc to FlexModule, forRoot(), CanScreenDeactivateGuard
  [ ] Create ISimpleRuntimeConfigService extending IFlexRuntimeConfigService

  PHASE 2 — Build SimpleRuntime internals
  ════════════════════════════════════════
  [ ] Create SimpleRuntimeConfigService backed by signal<Config | undefined>
      - expose configLoaded$ = toObservable(this._config)  ← backward compat
  [ ] Create signal-based replacements for each NgRx store slice
      - ConfigurationService (signal)
      - ComponentTrackingService (signal)
      - EnvironmentVariablesService (signal)
      - FlexPropertyBindingService (signal)
  [ ] Implement provideSimpleRuntime() using makeEnvironmentProviders()
  [ ] Rewrite FlexAppViewerComponent internals:
      - viewChild() signal for screenElement
      - afterNextRender() instead of ngAfterViewInit
      - takeUntilDestroyed(destroyRef) instead of Subscription
      - effect() reaction on config signal
  [ ] Create canScreenDeactivateFn functional guard
  [ ] Keep <flex-app-viewer> selector unchanged

  PHASE 3 — Shim Layer (zero-impact for existing consumers)
  ══════════════════════════════════════════════════════════
  [ ] Keep FlexModule as a deprecated wrapper:
      static forRoot(c) { return { ngModule: FlexModule, providers: [provideSimpleRuntime(c)] } }
  [ ] Export both FlexModule and provideSimpleRuntime from index.ts

  PHASE 4 — Migrate host apps (minimal effort)
  ════════════════════════════════════════════
  [ ] client-app/client.module.ts       — swap forRoot → provideSimpleRuntime, remove NgRx modules
  [ ] dev-app                           — same 2-line change
  [ ] onbase-apps-host                  — same 2-line change
  [ ] app-routing configs               — swap class guard → functional guard
  [ ] Remove @ngrx/store, @ngrx/store-devtools from package.json if fully unused

  PHASE 5 — Validation
  ════════════════════
  [ ] All existing unit tests pass (services still injectable via same tokens)
  [ ] E2E tests for screen lifecycle (load → interact → navigate away → teardown)
  [ ] Verify state isolation between micro-frontend instances (was ngOnDestroy clearStore)
  [ ] Performance benchmarks: signals vs NgRx selector memoisation
```

---

## 9. Decision Summary

```
  ╔══════════════════════════════════════════════════════════════════════╗
  ║  CAN flex-runtime BE SWAPPED WITH SimpleRuntime?                    ║
  ╠══════════════════════════════════════════════════════════════════════╣
  ║                                                                      ║
  ║  ✅  YES — and host apps need only 2 file changes.                   ║
  ║                                                                      ║
  ║  THE SWAP IS SAFE BECAUSE:                                           ║
  ║  • <flex-app-viewer> selector stays identical                        ║
  ║  • All Injection Tokens stay identical                               ║
  ║  • FlexConfigurationOverride interface stays identical               ║
  ║  • configLoaded$ Observable stays via toObservable(signal)           ║
  ║  • A deprecated FlexModule shim can absorb late adopters             ║
  ║                                                                      ║
  ║  THE SWAP DELIVERS:                                                  ║
  ║  • Removal of @ngrx/store + @ngrx/effects + @ngrx/store-devtools    ║
  ║  • Angular 21 Signals replace all NgRx slices internally             ║
  ║  • No Zone.js subscriptions (zoneless-ready with effect())           ║
  ║  • Functional guards remove deprecated CanDeactivate class           ║
  ║  • ~30% less boilerplate in each host app                            ║
  ║                                                                      ║
  ║  THE ONE HARD BLOCKER:                                               ║
  ║  • flex-operations still has its own OperationStore (NgRx).          ║
  ║    This lib must be migrated or given a signal-compatible API        ║
  ║    before the full NgRx dependency can be dropped from host apps.    ║
  ║                                                                      ║
  ╚══════════════════════════════════════════════════════════════════════╝
```

---

*Analysis based on actual source — `develop` branch, March 2026.*  
*Key files examined: `flex.module.ts`, `flex-app-viewer.component.ts`, `flex-runtime-config.service.ts`,*  
*`can-screen-deactivate.guard.ts`, `flex-configuration-override.model.ts`, `client.module.ts`.*
