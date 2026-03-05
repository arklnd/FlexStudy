# SimpleRuntime Migration — Impact & Effort Report

*Based on live code analysis — `develop` branch, 5 March 2026.*

---

## Executive Summary

| Dimension | Detail |
|---|---|
| **Total impacted files** | ~25 production files, ~15 spec files |
| **Mandatory host-app changes** | 3 apps × 1–2 files each |
| **Primary rewrite target** | `libs/flex-runtime` (~2,055 prod LOC) |
| **Hard blocker** | `libs/flex-operations` owns an NgRx `OperationStore` that `flex-runtime` dispatches into |
| **Effort estimate** | **7–10 engineering weeks** (with test time) across 5 phases |

---

## 1. Host Applications — Mandatory Changes

These are the bootstrapper apps that currently call `FlexModule.forRoot()` and wire `StoreModule`.

### `apps/client-app`

**Files changed: 1 mandatory, 2 additional**

| File | Change | Type |
|---|---|---|
| `client.module.ts` | `FlexModule.forRoot()` → `provideSimpleRuntime()` | MANDATORY |
| `client.module.ts` | Remove `StoreModule.forRoot()` + `StoreDevtoolsModule` (lines 3–4, 43–63) | REMOVAL |
| `app-routing.module.ts` | Guard handled via `flex-host` — no direct `canDeactivate` reference found in this app | NO CHANGE |

> **Note**: `app.component.ts` and two services (`user-settings.service.ts`, `shell-workview-controller-interaction.service.ts`) import `Store` from `@ngrx/store` directly — these are app-level NgRx usages *outside* `flex-runtime`, and must be audited separately. They are **not** removed by this migration unless the app itself drops NgRx entirely.

**Estimated effort: 0.5 days** (module wiring only)

---

### `apps/dev-app`

**Files changed: 1**

| File | Change | Type |
|---|---|---|
| `main.ts` | Remove `StoreModule.forRoot({})` + `EffectsModule.forRoot()` (lines 6–7, 30–31); add `provideSimpleRuntime()` | MANDATORY |

**Estimated effort: 0.5 days**

---

### `apps/onbase-apps-host`

**Files changed: 1**

| File | Change | Type |
|---|---|---|
| `main.ts` | `FlexModule.forRoot()` → `provideSimpleRuntime()`; remove `StoreModule.forRoot()` + `StoreDevtoolsModule` (lines 5–6, 37–49, 52–64) | MANDATORY |

**Estimated effort: 0.5 days**

---

### `apps/workflow-approval-mgmt`

**Not impacted by this migration.**

This app's `app.module.ts` uses `EffectsModule`/`StoreModule` to serve `wf-approval-mgmt-lib` state — completely independent of `flex-runtime`. It does not call `FlexModule.forRoot()`.

---

## 2. Library: `libs/flex-runtime` — Primary Rewrite Target

**Production LOC: 2,055 · Spec LOC: 3,025**

This is the core migration work. Every internal NgRx store is replaced with Angular Signals.

### 2A. NgRx Stores → Angular Signals (5 stores)

| Current Store File | Replacement | Effort |
|---|---|---|
| `configuration.store.ts` + `configuration.service.ts` | `signal<AppConfig>(...)` inside a service | Med |
| `component-tracking.service.ts` | Single `Map` signal (eliminates split-storage workaround with `immer`) | Med |
| `flex-runtime-config.store.ts` | `signal<Config \| undefined>` + `toObservable()` shim for `configLoaded$` | Med |
| `environment-variables.store.ts` | Signal in `EnvironmentVariablesService` | Low |
| `flex-property-binding.service.ts` | Signal + `effect()` replaces manual `controlSubscriptions` dictionary | High |

### 2B. Core Component & Guard Changes

| File | Change |
|---|---|
| `flex.module.ts` | Add `provideSimpleRuntime()` export; keep `FlexModule` as deprecated shim |
| `flex-app-viewer.component.ts` | `@ViewChild` → `viewChild()`, `ngAfterViewInit` → `afterNextRender()`, remove `Subscription` management, `store.dispatch(clearOperationStore)` → direct signal reset |
| `can-screen-deactivate.guard.ts` | Replace class-based `CanDeactivate` (deprecated in Angular 15, removed in Angular 17) with functional `canScreenDeactivateFn`; fix documented navigation-history corruption bug |
| `flex-runtime-config.service.ts` | Signal-backed; expose `configLoaded$ = toObservable(this._config)` for backward compat |
| `form-to-screen-registry.service.ts` | Fix constructor subscription leak (tracked as TODO SBPFAB-153) |
| `index.ts` | Export `provideSimpleRuntime`, `canScreenDeactivateFn` alongside deprecated exports |

### 2C. Dependency Removals (inside `flex-runtime`)

| Package | Current Usage | Action |
|---|---|---|
| `@ngrx/store` | 5 store files, 3 service files | **Remove** |
| `@ngrx/effects` | `flex.module.ts` — `EffectsModule.forRoot()` | **Remove** |
| `@ngrx/store-devtools` | `flex-test.module.ts` | **Remove** |
| `immer` | `configuration.store.ts`, `environment-variables.store.ts`, `configuration-store.models.ts`, `flex-control-init.service.spec.ts` | **Remove** |
| `uuid` | `flex-control-init.service.ts` — `v4 as generateUuid` | Replace with `crypto.randomUUID()` |

### 2D. API Cleanup (Optional but low-risk)

| File | Issue | Fix |
|---|---|---|
| `flex-broadcast.service.ts` | `broadcastRecieved$` (public typo on interface) | Add `broadcastReceived$` alias; deprecate old spelling |
| `flex-external-provider.interface.ts` | Lives in a folder literally named `temp/` | Promote to first-class path in new release |

**Estimated effort: 4–5 weeks** (implementation + unit test rewrites for 3,025 spec LOC)

---

## 3. Library: `libs/flex-host` — Minor Update

**Production LOC: 93 · Spec LOC: 48**

| File | Change | Type |
|---|---|---|
| `flex-host.component.ts` | Line 21: `canDeactivate: [CanScreenDeactivateGuard]` → `canDeactivate: [canScreenDeactivateFn]` | MANDATORY (1 line) |

**Estimated effort: 0.5 days**

---

## 4. Library: `libs/flex-operations` — Blocker / Parallel Migration

**Production LOC: 2,736 · Tier 3 — MODERATE re-engineering risk**

This is the **hard blocker** identified in the migration plan. `flex-runtime`'s `FlexAppViewerComponent` and `FlexPropertyBindingService` both dispatch into and select from `OperationStore`.

| File | NgRx Coupling |
|---|---|
| `operation-store.ts` | Full NgRx feature store (`createFeature`, `createReducer`, `EffectsModule`) |
| `operation-state-store.proxy.ts` | `Actions`, `createEffect`, `ofType`, `Store` |
| `operation-execution-utilities.ts` | `Store` injection |

Until `flex-operations` is migrated (or given a signal-compatible facade), host apps **cannot drop NgRx entirely**, even after `flex-runtime` is updated.

**Options:**

1. **Preferred**: Migrate `flex-operations` to signals in parallel with `flex-runtime` (adds ~2 weeks)
2. **Stop-gap**: Create an `IOperationStore` interface that both the NgRx and signal implementations satisfy, letting `flex-runtime` call through the interface without caring which backend is used

**Estimated effort: 2–3 weeks** (if migrated; ~1 week for compatibility facade)

---

## 5. Test Infrastructure

| Scope | Impact |
|---|---|
| `flex-test.module.ts` | Rewrite to use `provideSimpleRuntime()` instead of all 5 `getModuleImports()` calls + `StoreDevtoolsModule` |
| `flex-operations-test.module.ts` | Rewrite — uses `StoreModule`, `EffectsModule`, `StoreDevtoolsModule` |
| All 3,025 `flex-runtime` spec lines | Jasmine spies on `store.dispatch` / `store.select` → spies on signal/service methods |

---

## 6. Effort Summary

| Phase | Scope | Effort |
|---|---|---|
| **Phase 1** — Preparation | Audit NgRx imports, add deprecation JSDoc, define interfaces | 1 week |
| **Phase 2** — SimpleRuntime internals | 5 signal stores, `provideSimpleRuntime()`, refactored components, functional guard, API cleanup | 4–5 weeks |
| **Phase 3** — flex-operations unblock | Signal migration or compatibility facade for `OperationStore` | 1–2 weeks |
| **Phase 4** — Host app wiring | 3 apps × ~5-line change, guard swap | 2 days |
| **Phase 5** — Validation | Unit tests, E2E screen lifecycle, state isolation, perf benchmarks | 1–2 weeks |
| | **Total** | **7–10 weeks** |

---

## 7. What Does NOT Change (Zero Host-App Risk)

- `<flex-app-viewer>` selector — stays identical
- `FLEX_RUNTIME_CONFIG_SERVICE`, `FLEX_BROADCAST_SERVICE`, `FLEX_EXTERNAL_PROVIDER` injection tokens — all preserved
- `FlexExternalProvider` interface — unchanged
- `FlexConfigurationOverride` config shape — unchanged
- `configLoaded$` Observable — preserved via `toObservable(signal)`
- `FlexModule.forRoot()` — kept as a deprecated shim so third-party consumers outside this repo are unaffected during the transition

---

*Generated from live codebase analysis — `develop` branch, 5 March 2026.*
