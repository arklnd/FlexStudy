# `onbase-apps-shell` — Architecture Deep Dive

> **Audience**: Engineers building or maintaining applications that run inside the OnBase Apps shell chrome.  
> **Package**: `@hyland/onbase-apps-shell`  
> **LOC**: 218 lines (TS src) · 212 spec · 63 HTML · 51 SCSS  
> **Usage score**: 6 import-lines across the repo · Tier 4 — Low Effort  
> **Companion document**: See [`libs/ARCHITECTURE.md`](../ARCHITECTURE.md) for the full cross-library picture.

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [The Big Picture](#2-the-big-picture)
3. [Public API](#3-public-api)
4. [Component Architecture](#4-component-architecture)
   - [OBAShellComponent](#41-obashellcomponent)
   - [OBAShellHeaderComponent](#42-obashellheadercomponent)
   - [OBAShellViewComponent](#43-obashellviewcomponent)
5. [OBAShellService](#5-obashellservice)
6. [OnBaseAppsShellModule](#6-onbaseappsshellmodule)
7. [Data Flow](#7-data-flow)
8. [Banner Animation](#8-banner-animation)
9. [Design Principles](#9-design-principles)

---

## 1. Purpose

`onbase-apps-shell` provides the **application-level chrome** for OnBase Apps — the shared header toolbar, feedback banner, and routing guard wiring that wrap individually loaded micro-frontend applications.

It answers the question: "What does the outer frame of an OnBase application look like?" — the toolbar with the Hyland logo, the title, the settings menu, the Content Intelligence Connector link, the logout button, and the animated feedback banner below the header.

---

## 2. The Big Picture

```
  OBAShellComponent  (onbase-apps-shell)
  ┌─────────────────────────────────────────────────────────────────────┐
  │                                                                     │
  │  ┌──────────────────────────────────────────────────────────────┐   │
  │  │  OBAShellHeaderComponent                                     │   │
  │  │  • Logo slot (ng-template)                                   │   │
  │  │  • Title (headerTitle$ | async)                              │   │
  │  │  • Content Intelligence Connector icon link                  │   │
  │  │  • Settings menu icon                                        │   │
  │  │  • Logout button → logOutClick EventEmitter                  │   │
  │  │  • [DEV] Pseudo-localization toggle                          │   │
  │  └──────────────────────────────────────────────────────────────┘   │
  │                                                                     │
  │  ┌──────────────────────────────────────────────────────────────┐   │
  │  │  HyFeedbackBanner (animated banner, @bannerExpansion)        │   │
  │  │  bannerMessage$ | async                                      │   │
  │  └──────────────────────────────────────────────────────────────┘   │
  │                                                                     │
  │  <ng-content>   ← host app mounts its router-outlet here            │
  └─────────────────────────────────────────────────────────────────────┘
  
  OBAShellViewComponent (bridge: host app → OBAShellService)
  ┌───────────────────────────────────────────────────────────────────┐
  │  @Input() headerTitle       → OBAShellService.setHeaderTitle()    │
  │  @Input() contentIntelligenceConnector → setContentIntelligence() │
  └───────────────────────────────────────────────────────────────────┘
  
  OBAShellService (RxJS bridge)
  ┌───────────────────────────────────────────────────────────────────┐
  │  bannerMessage$                Subject<string>                    │
  │  contentIntelligenceConnector$ Subject<string> (delay(0))         │
  │  headerTitle$                  Subject<string> (delay(0))         │
  └───────────────────────────────────────────────────────────────────┘
```

---

## 3. Public API

```
  onbase-apps-shell/src/index.ts
  ├── OBAShellComponent        — main shell wrapper component
  ├── OBAShellHeaderComponent  — the header toolbar (re-exported via module)
  ├── OBAShellViewComponent    — @Input bridge from host app into shell service
  ├── OnBaseAppsShellModule    — NgModule with BlockingService + guard helpers
  └── OBAShellService          — observable streams for shell chrome state
```

---

## 4. Component Architecture

### 4.1 OBAShellComponent

**Selector**: `<oba-shell>`  
**File**: `lib/components/shell/shell.component.ts`  
**Encapsulation**: `ViewEncapsulation.None`

The root shell container. Listens to `_shellService.bannerMessage$` and renders the banner when a message is emitted. Uses `ng-content` for slot-based content projection so host apps inject their own router outlets.

**Inputs:**

| Input | Type | Description |
|-------|------|-------------|
| `customIcons` | `TemplateRef<unknown>` | Extra icon buttons in the header toolbar |
| `logo` | `TemplateRef<unknown>` | Application logo template |
| `hidden` | `boolean \| null` | Hides the entire shell chrome |
| `showSettings` | `boolean` | Shows/hides the settings gear icon |
| `showPsuedoLocalizationIcon` | `boolean` | Dev-only: shows pseudo-localization toggle |

**Outputs:**

| Output | Description |
|--------|-------------|
| `logOutClick` | Emitted when user clicks the logout button |

---

### 4.2 OBAShellHeaderComponent

**Selector**: `<oba-shell-header>` (internal to `OBAShellComponent`)  
**File**: `lib/components/shell-header/shell-header.component.ts`

Renders the Material toolbar with:
- Application logo (via `ng-template` slot)
- Application title (`headerTitle$ | async`)
- Content Intelligence Connector icon (shows if `contentIntelligenceConnector` is set)
- Settings icon (shows if `showSettings`)
- Logout button
- `[DEV]` Pseudo-localization toggle (`showPsuedoLocalizationIcon`)

Uses `TranslationPipe` (`obTranslate`) from `onbase-web-server-core` for button labels.

---

### 4.3 OBAShellViewComponent

**Selector**: `<oba-shell-view>`  
**File**: `lib/components/shell-view/shell-view.component.ts`  
**Change Detection**: `OnPush`

This component solves an architectural challenge: the host application knows its title and Content Intelligence Connector URL, but the shell chrome (`OBAShellComponent`) is rendered separately (often as a sibling in the component tree). This component acts as the **bridge** — an `@Input()` portal into `OBAShellService`.

```typescript
// In host app template:
<oba-shell-view
  [headerTitle]="'My Application Title'"
  [contentIntelligenceConnector]="ciConnectorUrl">
</oba-shell-view>

<oba-shell [logo]="logoTmpl">
  <router-outlet></router-outlet>
</oba-shell>
```

When `@Input()` values change, `ngOnChanges` calls the shell service to update observable streams.

---

## 5. OBAShellService

**File**: `lib/services/shell.service.ts`  
**Provided**: `{ providedIn: 'root' }` — singleton

Central reactive state hub for shell chrome state:

```
  Subject Sources                     Consumers
  ─────────────────                   ──────────────────────────────────
  bannerMessageSubject                bannerMessage$ (raw, no delay)
    ← setBannerMessage(msg)             → OBAShellComponent banner

  contentIntelligenceConnectorSubject contentIntelligenceConnector$ (delay(0))
    ← setContentIntelligenceConnector()  → OBAShellHeaderComponent link

  headerTitleSubject                  headerTitle$ (delay(0))
    ← setHeaderTitle(title)             → OBAShellHeaderComponent title
                                        → Angular Title service (document title)
```

The `delay(0)` on `contentIntelligenceConnector$` and `headerTitle$` prevents Angular `ExpressionChangedAfterChecked` errors when the inputs are set in the same change detection cycle that renders the header.

---

## 6. OnBaseAppsShellModule

**File**: `lib/onbase-apps-shell.module.ts`

An NgModule that provides `BlockingService` from `@hyland/onbase-web-server` and exposes a static factory for route guard configuration:

```typescript
OnBaseAppsShellModule.includeBlockingGuards()
// Returns:
{
  canActivate: [
    BlockingService,
    FlexRouterNavigationFacilitatorService.isRouterFinishedLoadingGuard
  ]
}
```

This guard combination ensures that:
1. **`BlockingService`** — prevents navigation if the UI is in a "blocked" state (e.g., a process is running)
2. **`isRouterFinishedLoadingGuard`** — prevents premature navigation before the Flex router finishes lazy-loading

Usage in an app's routing:

```typescript
const routes: Routes = [
  {
    path: '',
    ...OnBaseAppsShellModule.includeBlockingGuards(),
    children: [...]
  }
];
```

---

## 7. Data Flow

```
  Host App Component
  └── @Input() data changes
        │
        ▼
  OBAShellViewComponent.ngOnChanges()
        │
        ├── headerTitle changed?
        │     └── OBAShellService.setHeaderTitle(title)
        │           ├── headerTitleSubject.next(title)
        │           └── Title.setTitle(title)   [browser tab title]
        │
        └── contentIntelligenceConnector changed?
              └── OBAShellService.setContentIntelligenceConnector(url)
                    └── contentIntelligenceConnectorSubject.next(url)
  
  OBAShellComponent (async pipe)
  ├── _shellService.bannerMessage$ → HyFeedbackBanner [*ngIf="banner"]
  └── delegates to OBAShellHeaderComponent
        ├── _shellService.headerTitle$ → title displayed in toolbar
        └── _shellService.contentIntelligenceConnector$ → CI icon visibility
```

---

## 8. Banner Animation

The feedback banner uses an Angular animation defined in `shell.component.ts`:

```typescript
export const bannerExpansionAnimation = trigger('bannerExpansion', [
  transition(':enter', [
    style({ height: '0px' }),
    animate('500ms cubic-bezier(0.4, 0.0, 0.2, 1)', style({ height: '*' }))
  ]),
]);
```

This creates a smooth slide-down effect when a `bannerMessage$` is emitted. The banner uses Material Design's standard easing curve.

---

## 9. Design Principles

| Principle | Implementation |
|-----------|---------------|
| **RxJS bridge over @Output chains** | `OBAShellService` decouples header state from deep input passing |
| **Slot-based composition** | `ng-content` + `TemplateRef` inputs keep the shell generic |
| **Change-detection safe** | `delay(0)` prevents `ExpressionChangedAfterChecked` in all streams |
| **Static guard factory** | `includeBlockingGuards()` gives one consistent guard config to all routes |
| **Separation of concerns** | View bridge (`OBAShellViewComponent`) vs. chrome container (`OBAShellComponent`) are separate |

---

*Generated from codebase analysis — `develop` branch, March 2026.*
