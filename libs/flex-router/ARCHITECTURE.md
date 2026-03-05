# `flex-router` — Architecture Deep Dive

> **Audience**: Engineers working on screen navigation, outlet rendering, or breakpoint-based routing.  
> **Package**: `@hyland/flex-router`  
> **LOC**: 3,116 lines (TS src) · 3,520 spec · 5 HTML · 11 SCSS  
> **Usage score**: 35 import-lines across the repo · Tier 2 — High Effort  
> **Companion document**: See [`libs/ARCHITECTURE.md`](../ARCHITECTURE.md) for the full cross-library picture.

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [The Big Picture](#2-the-big-picture)
3. [Architecture Overview](#3-architecture-overview)
4. [Key Components](#4-key-components)
   - [FlexRouterOutletComponent](#41-flexrouteroutletcomponent)
   - [FlexScreenComponent + FlexScreenElement](#42-flexscreencomponent--flexscreenelement)
5. [Core Services](#5-core-services)
   - [FlexRouterService](#51-flexrouterservice)
   - [FlexRouterTreeService](#52-flexroutertreeservice)
   - [RouterParameterService](#53-routerparameterservice)
   - [FlexRouterNavigationFacilitatorService](#54-flexrouternavigationfacilitatorservice)
   - [FlexRouterConfigService](#55-flexrouterconfigservice)
   - [ScreenTrackingService + OutletTrackingService](#56-screentrackingservice--outlettrackingservice)
   - [FocusService](#57-focusservice)
6. [Navigation Flow](#6-navigation-flow)
7. [Breakpoint Routing](#7-breakpoint-routing)
8. [CanNavigateAway Guard System](#8-cannavigateaway-guard-system)
9. [Directives](#9-directives)
10. [DI Token Map](#10-di-token-map)
11. [Design Principles](#11-design-principles)

---

## 1. Purpose

`flex-router` is the **screen rendering and navigation engine** for the Flex runtime. It answers the question: "Given the current URL, which Flex screen should be rendered in which outlet, at which responsive breakpoint?"

Standard Angular Router alone is insufficient for Flex's requirements because:
- Screens are defined in _configuration_, not in `@NgModule` routing arrays
- The same route may need different router outlets at different screen widths (breakpoints)
- Outlets are themselves Flex components registered at runtime, not fixed `<router-outlet>` tags in templates
- Navigation must be coordinated across a tree of outlets, checking each for dirty state before proceeding

`flex-router` builds a parallel routing layer on top of Angular Router that satisfies all these requirements.

---

## 2. The Big Picture

```
  Angular Router (URL changes)
  ┌────────────────────────────────────────────────────────┐
  │  NavigationEnd event                                   │
  └────────────────────────────┬───────────────────────────┘
                               │
                               ▼
  FlexRouterService
  ┌────────────────────────────────────────────────────────┐
  │  Subscribes to NavigationEnd                           │
  │  Reads current URL → getRootPathForApp()               │
  │  updateRouteTree(rootPath)                             │
  │      ├── FlexRouterTreeService.buildTree(screenConfig) │
  │      │     computes: screen → outlet mapping           │
  │      └── For each <flex-router-outlet> in tree:        │
  │            outlet.activateRoute(screenId)              │
  └──────────────────────────┬─────────────────────────────┘
                             │
                             ▼
  FlexRouterOutletComponent.activateRoute(screenId)
  ┌────────────────────────────────────────────────────────┐
  │  Creates (or reuses) FlexScreenComponent for screenId  │
  │  Attaches it to the outlet's host element              │
  └──────────────────────────┬─────────────────────────────┘
                             │
                             ▼
  FlexScreenComponent
  ┌────────────────────────────────────────────────────────┐
  │  Renders htmlContentTemplate (from ScreenConfiguration)│
  │  Mounts Flex components, wires lifecycle hooks         │
  └────────────────────────────────────────────────────────┘
```

---

## 3. Architecture Overview

```
  flex-router/src/lib/
  ├── components/
  │   ├── flex-router-outlet/   FlexRouterOutletComponent  — renders a screen
  │   └── flex-screen/          FlexScreenComponent        — represents one screen
  │                             FlexScreenElement          — Web Component interface
  ├── directives/
  │   ├── flex-router-link.directive.ts       — [flexRouterLink]
  │   └── flex-router-link-active.directive.ts — [flexRouterLinkActive]
  ├── interfaces/                DI tokens + service contracts
  ├── mocks/                     Test doubles for all services
  ├── models/
  │   └── tracking-service/     TrackedScreen, TrackedOutlet
  ├── services/
  │   ├── router/
  │   │   ├── flex-router.service.ts           FlexRouterService (main coordinator)
  │   │   ├── router-config/                   FlexRouterConfigService
  │   │   ├── router-navigation-facilitator/   FlexRouterNavigationFacilitatorService
  │   │   └── router-tree/                     FlexRouterTreeService (builds outlet tree)
  │   ├── route-parameter-service/             RouterParameterService
  │   ├── tracking-service/                    Screen + Outlet tracking
  │   ├── abstract-screen.service.ts           Base class for screen lifecycle
  │   ├── focus.service.ts                     Input focus management
  │   └── navigation-facilitator.service.ts    Coordinates canNavigateAway checks
  ├── pipes/
  │   └── safeHtml.pipe.ts                     Bypasses Angular DomSanitizer for templates
  ├── flex-router.module.ts                    NgModule registering all services
  └── test-utils/
```

---

## 4. Key Components

### 4.1 FlexRouterOutletComponent

**Selector**: Custom element (`<flex-router-outlet>`) — registered as a Web Component  
**Encapsulation**: `ViewEncapsulation.ShadowDom`

The router outlet is the **hole in the page** where a screen is placed. Each outlet:
1. Self-registers with `FlexRouterService` when Flex initializes it (after `ComponentTrackingService` sees it)
2. Exposes `activateRoute(screenId)` — called by `FlexRouterService` when the URL changes
3. Tracks its current `screenId` to avoid re-rendering unchanged screens
4. Checks `canNavigateToNewRoute()` before allowing navigation away (dirty state check)
5. Cleans up on `ngOnDestroy` — unregisters from tracking service

**Key properties:**

| Property | Description |
|----------|-------------|
| `id` | Flex control ID (from DOM attr `flex-control-id`) |
| `controlInstanceId` | Unique per-instance ID |
| `outletInstanceId` | Auto-generated UUID per render cycle |
| `screenId` | Currently active screen ID |
| `isRoutingLoading$` | `BehaviorSubject<boolean>` for loading indicator |

---

### 4.2 FlexScreenComponent + FlexScreenElement

`FlexScreenComponent` renders the HTML content template for a screen and manages its full lifecycle (init hooks, destroy hooks, child component wiring).

`FlexScreenElement` is the custom element interface that the router outlet interacts with programmatically (call `canDeactivate()`, `destroyScreen()`, etc.).

---

## 5. Core Services

### 5.1 FlexRouterService

Implements `IFlexRouterService` (`FLEX_ROUTER_SERVICE` token).

**Responsibilities:**
- Subscribe to Angular `NavigationEnd` events
- Call `FlexRouterTreeService` to compute the new outlet→screen mapping
- Coordinate activation of all outlets in the tree
- Track loading state via `isRoutingLoading$` Observable

**Loading state** is reference-counted:
```typescript
routingLoadingStarted()   // ++numLoading → emits true
routingLoadingFinished()  // --numLoading → emits false when count = 0
```

This allows multiple outlets loading in parallel to correctly reflect the overall loading state.

---

### 5.2 FlexRouterTreeService

Computes the **tree of outlet → screen assignments** from the current URL and `RouteConfiguration[]` in the config:

```
  Input: current URL path + ScreenConfiguration[] (with RouteConfiguration[])
  Output: tree of { outlet → screenId } assignments

  Algorithm:
  1. Parse URL into path segments
  2. Find the ScreenConfiguration for the current path
  3. For each RouteConfiguration in that screen:
       a. Match breakpoint (current CSS media query → which outlet ID wins)
       b. Create OutletTreeNode { outletId, screenId, children: [] }
  4. Recursively resolve child routes (nested routing)
```

---

### 5.3 RouterParameterService

Implements `IRouterParameterService` (`FLEX_ROUTER_PARAMETER_SERVICE` token).

Provides access to query string, route, and fragment parameters from the current URL. Backed by an NgRx store slice (`flex-router-parameter.store.ts`) so parameter changes are observable.

```typescript
interface IRouterParameterService {
  getRouteParams(): RouteParamDictionary;
  getQueryParams(): RouteParamDictionary;
  routeParamsChanged$: Observable<RouterParameterChanged>;
  queryParamsChanged$: Observable<RouterParameterChanged>;
}
```

---

### 5.4 FlexRouterNavigationFacilitatorService

Implements `IFlexNavigationFacilitator`. Used by `flex-host` (provides `CanScreenDeactivateGuard`) and by `onbase-apps-shell` to block navigation.

Key method:
```typescript
static isRouterFinishedLoadingGuard: CanActivateFn
// Returns false (blocks) while isRoutingLoading$ is true
```

Also exposes:
- `canNavigateAwayFromScreen(screen)` — delegates to the screen's registered `CanNavigateAwayFromScreenResolver`

---

### 5.5 FlexRouterConfigService

Implements `IFlexRouterConfigService` (`FLEX_ROUTER_CONFIG_SERVICE` token).

Provides the router with access to config-level routing data:
```typescript
getRootPathForApp(): string
getScreenConfiguration(screenId: string): ScreenConfiguration
getRouteConfigurations(screenId: string): RouteConfiguration[]
```

---

### 5.6 ScreenTrackingService + OutletTrackingService

Two complementary registries:

**`ScreenTrackingService`** (`SCREEN_TRACKING_SERVICE` token):
- Maps `screenInstanceId → screenId`
- Maps `screenInstanceId → FlexScreenElement instance`
- `TrackedScreen` model: `{ id, systemName, instanceId, element }`

**`OutletTrackingService`** (`OUTLET_TRACKING_SERVICE` token):
- Maps `outletInstanceId → TrackedOutlet`
- Provides `getOutletInformationByID()`
- `TrackedOutlet` model: `{ id, screenElement, ... }`

---

### 5.7 FocusService

Implements `IFocusService` (`FOCUS_SERVICE` token). Manages focus restoration after navigation:
- Saves focused element before navigation
- Restores focus after screen is activated
- Prevents focus thrashing during rapid navigation

---

## 6. Navigation Flow

```
  User navigates (URL changes via Angular Router)
         │
         ▼
  Angular NavigationEnd event fires
         │
         ▼
  FlexRouterService.updateRouteTree(rootPath)
         │
         ├── Check all registered outlets for canNavigateToNewRoute()
         │     └── NavigationFacilitatorService.canNavigateAwayFromScreen()
         │           └── Calls each CanNavigateAwayFromScreenResolver
         │                 (e.g. dirty form check)
         │
         └── FlexRouterTreeService.buildTree(currentPath)
               └── returns: Map<outletId, screenId>
         │
         ▼
  For each outlet in tree:
         outlet.activateRoute(screenId)
               │
               ├── if screenId changed:
               │     screenService.createScreenAndAttachToElement(hostEl, screenId, outlet)
               │     ├── Destroys old screen (if any)
               │     ├── Creates new FlexScreenComponent dynamically
               │     ├── Sets screen config, fires ScreenInit lifecycle hooks
               │     └── Attaches to outlet's host element
               │
               └── if screenId unchanged: no-op
```

---

## 7. Breakpoint Routing

A single `RouteConfiguration` can specify different outlets for different CSS breakpoints:

```typescript
// From flex-config:
interface RouteBreakpointConfiguration {
  breakpoint: string;               // e.g. 'gt-sm', 'lt-md', or '*' for default
  flexRouterOutletControlId: string;
}
```

`FlexRouterTreeService` evaluates which breakpoint matches the current viewport using CSS media queries and selects the corresponding outlet ID. This allows Flex applications to show different screen layouts (e.g., a sidebar on desktop, a full-page drawer on mobile) without separate route configurations.

---

## 8. CanNavigateAway Guard System

```
  CAN_NAVIGATE_AWAY_FROM_SCREEN_RESOLVERS token
  ├── Array of CanNavigateAwayFromScreenResolver instances
  └── Each resolver: canNavigateAway(screen: FlexScreenElement): Promise<boolean>

  NavigationFacilitatorService.canNavigateAwayFromScreen(screen)
  ├── Calls all resolvers in sequence
  ├── If any returns false → blocks navigation
  └── Returns true only if all resolvers agree

  flex-host provides CanScreenDeactivateGuard as one resolver
```

This system allows multiple concerns (dirty forms, confirmation dialogs, in-flight operations) to each independently block navigation.

---

## 9. Directives

| Directive | Attribute | Behavior |
|-----------|-----------|---------|
| `FlexRouterLink` | `[flexRouterLink]` | Like Angular's `[routerLink]` but for Flex screen paths |
| `FlexRouterLinkActive` | `[flexRouterLinkActive]` | Adds a CSS class when the linked route is active |

---

## 10. DI Token Map

```
  Token                                Interface                        Provided by
  ────────────────────────────────     ───────────────────────────      ──────────────────────
  FLEX_ROUTER_SERVICE                  IFlexRouterService               FlexRouterModule
  FLEX_ROUTE_GUARDS_TOKEN              IFlexRouteGuards                 flex-host / consumer
  FLEX_ROUTER_CONFIG_SERVICE           IFlexRouterConfigService         FlexRouterModule
  FLEX_ROUTER_NAVIGATION_FACILITATOR_SERVICE  IFlexNavigationFacilitator  FlexRouterModule
  FLEX_ROUTER_PARAMETER_SERVICE        IRouterParameterService          FlexRouterModule
  SCREEN_SERVICE                       IScreenService                   flex-runtime
  OUTLET_TRACKING_SERVICE              IOutletTrackingService           FlexRouterModule
  SCREEN_TRACKING_SERVICE              IScreenTrackingService           FlexRouterModule
  FOCUS_SERVICE                        IFocusService                    FlexRouterModule
  CAN_NAVIGATE_AWAY_FROM_SCREEN_RESOLVERS  CanNavigateAwayFromScreenResolver[]  flex-host / consumer
```

---

## 11. Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Config-driven routes** | No route arrays — all routing from `RouteConfiguration[]` in `FlexRuntimeConfiguration` |
| **Reference-counted loading** | `routingLoadingStarted/Finished()` pattern handles parallel outlet loads |
| **Cooperative navigation guards** | `CAN_NAVIGATE_AWAY_FROM_SCREEN_RESOLVERS` multi-resolver system |
| **Responsive breakpoints** | Single route config → different outlets at different viewport widths |
| **Shadow DOM isolation** | `FlexRouterOutletComponent` uses `ViewEncapsulation.ShadowDom` |
| **Lazy deactivation** | Old screen not destroyed until new screen is confirmed and ready |

---

*Generated from codebase analysis — `develop` branch, March 2026.*
