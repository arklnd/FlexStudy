# `onbase-web-server-core` — Architecture Deep Dive

> **Audience**: Engineers integrating Angular code with the OnBase Web Client JavaScript layer.  
> **Package**: `@hyland/onbase-web-server-core`  
> **LOC**: 468 lines (TS src) · 47 spec · 0 HTML · 0 SCSS  
> **Usage score**: 117 import-lines across the repo · Tier 1 — CANNOT REMOVE  
> **Companion document**: See [`libs/ARCHITECTURE.md`](../ARCHITECTURE.md) for the full cross-library picture.

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [The Big Picture](#2-the-big-picture)
3. [Public API Summary](#3-public-api-summary)
4. [The Window Bridge Pattern](#4-the-window-bridge-pattern)
5. [Service Interface Catalog](#5-service-interface-catalog)
6. [Enums Catalog](#6-enums-catalog)
7. [TranslationPipe](#7-translationpipe)
8. [Mock Implementations](#8-mock-implementations)
9. [DI Token Map](#9-di-token-map)
10. [Design Principles](#10-design-principles)

---

## 1. Purpose

`onbase-web-server-core` is the **bridging layer between Angular DI and the OnBase Web Client's JavaScript globals**. 

The OnBase Web Client is a legacy ASP.NET application that exposes services as `window.*` globals (e.g., `window.OBStorage`, `window.SessionChecker`, `window.UnloadManager`). Angular code cannot safely use `window.*` globals directly — they cannot be mocked in tests, they break SSR, and they create hidden coupling.

This library solves this by:
1. Defining `InjectionToken` constants for each legacy service
2. Defining TypeScript `interface` contracts for each service
3. Providing mock implementations for every service
4. Exposing a typed `WebClientWindow` interface that replaces unsafe `(window as any)` casts

---

## 2. The Big Picture

```
  OnBase Web Client (AppHost.aspx)
  ┌─────────────────────────────────────────────────────────┐
  │  window.OBStorage       — user settings storage         │
  │  window.Page            — page variable lookup (LRM)    │
  │  window.SessionChecker  — session / logout              │
  │  window.UnloadManager   — before-unload lifecycle       │
  │  window.WindowsManager  — child window management       │
  │  window.DataValidation  — locale-aware data formatting  │
  │  window.Messenger       — pub/sub event bus             │
  │  window.$OB             — utility functions             │
  │  window.__WEB_CLIENT_APP_HOST_CONFIG__                  │
  └───────────────────────────────┬─────────────────────────┘
                                  │  (global JS objects)
                                  ▼
  ┌────────────────────────────────────────────────────────────┐
  │  onbase-web-server-core                                    │
  │                                                            │
  │  Defines:                                                  │
  │   WebClientWindow interface  —  typed window alias         │
  │   InjectionTokens for each global service                  │
  │   TypeScript interfaces for each global service            │
  │   Mock implementations of all interfaces                   │
  │                                                            │
  │  Exports:                                                  │
  │   WEB_CLIENT_WINDOW, WEB_CLIENT_APP_HOST_CONFIG tokens     │
  │   OB_STORAGE, PAGE, SESSION_CHECKER, UNLOAD_MANAGER,       │
  │   WINDOWS_MANAGER, DATA_VALIDATION tokens                  │
  │   ViewerType enum, WorkItemType enum                       │
  │   TranslationPipe (obTranslate)                            │
  └───────────────────────────────┬────────────────────────────┘
                                  │  (Angular DI)
                                  ▼
  ┌────────────────────────────────────────────────────────────┐
  │  onbase-web-server / wv-components / shared-components     │
  │  Inject tokens, cast window → WebClientWindow, use mocks   │
  └────────────────────────────────────────────────────────────┘
```

---

## 3. Public API Summary

```
  onbase-web-server-core/src/index.ts
  ├── Tokens
  │   ├── WEB_CLIENT_WINDOW              InjectionToken<WebClientWindow>
  │   ├── WEB_CLIENT_APP_HOST_CONFIG     InjectionToken<WebClientAppHostConfig>
  │   ├── OB_STORAGE                     InjectionToken<IOBStorageService>
  │   ├── PAGE                           InjectionToken<IPageService>
  │   ├── SESSION_CHECKER                InjectionToken<ISessionCheckerService>
  │   ├── UNLOAD_MANAGER                 InjectionToken<IUnloadManagerService>
  │   ├── DATA_VALIDATION                InjectionToken<IDataValidation>
  │   └── WINDOWS_MANAGER                InjectionToken<IWindowsManagerService>
  ├── Interfaces
  │   ├── WebClientWindow                typed alias for the global window object
  │   ├── WebClientAppHostConfig         AppHost.aspx injected config bag
  │   ├── IOBStorageService              user settings storage
  │   ├── IPageService                   page variable / LRM access
  │   ├── ISessionCheckerService         logout
  │   ├── IUnloadManagerService          before-unload lifecycle validators/finalizers
  │   └── IWindowsManagerService         child window management
  │   └── OnBaseWebServerUnloadableComponentInterface  canUnload/unload contract
  ├── Enums
  │   ├── ViewerType                     rendering context (Web, Unity, ObjectPop, ...)
  │   └── WorkItemType                   workflow content type
  └── Pipe
      └── TranslationPipe  (@Pipe name: 'obTranslate')
```

---

## 4. The Window Bridge Pattern

Every consuming component follows the same Angular DI pattern to safely access a Web Client global:

```typescript
// In the consuming app's AppModule or component providers:
providers: [
  {
    provide:  WEB_CLIENT_WINDOW,
    useValue: window as unknown as WebClientWindow
  },
  {
    provide:  OB_STORAGE,
    useFactory: (win: WebClientWindow) => win.OBStorage,
    deps: [WEB_CLIENT_WINDOW]
  }
]
```

```typescript
// In the consuming service / component:
constructor(@Inject(OB_STORAGE) private obStorage: IOBStorageService) {}

this.obStorage.getItem(this.obStorage.Keys.Viewer.UserSettings);
```

```typescript
// In tests — no window needed:
providers: [
  { provide: OB_STORAGE, useClass: MockOBStorageService }
]
```

This pattern ensures:
- **Zero window usage in component code** — only interfaces
- **Complete testability** — swap real window for mock with one provider
- **Type safety** — `WebClientWindow` interface catches missing properties

---

## 5. Service Interface Catalog

### `IPageService` (`PAGE` token)
Wraps `window.Page` — used to look up translated strings from the OnBase LRM (Language Resource Manager).

```typescript
interface IPageService {
  Get<T = string>(variable: string): T;
}
```

`Page.Get('ob_translations')` returns the full dictionary of translated UI strings.

---

### `IOBStorageService` (`OB_STORAGE` token)
Wraps `window.OBStorage` — typed key/value store for user preferences that survive across sessions.

```typescript
interface IOBStorageService {
  Keys: {
    Viewer: { UserSettings, Fit, ScaleToGray },
    Document: { EnableBrowserPdfViewer }
  };
  ClientSettingsGroup: { General: { UserThemePreference } };
  getItem(key: OBStorageKey, isTransient?: boolean): string | null;
  setItem(key: OBStorageKey, value: string, isTransient?: boolean): void;
  removeItem(key: OBStorageKey, isTransient?: boolean): void;
}
```

---

### `ISessionCheckerService` (`SESSION_CHECKER` token)
Wraps `window.SessionChecker`.

```typescript
interface ISessionCheckerService {
  logOut(): void;
}
```

---

### `IUnloadManagerService` (`UNLOAD_MANAGER` token)
Wraps `window.UnloadManager` — controls the page unload lifecycle.

```typescript
interface IUnloadManagerService {
  RegisterValidator(validator: BeforeUnloadValidator, invokeOnlyOnUnload?: boolean): void;
  RegisterFinalizer(finalizer: UnloadFinalizer, autoInvoke?: boolean, shouldExecuteLast?: boolean): void;
  DeferredCleanup(config: {
    e?: Event; withPrompt?: boolean; customMessage?: string;
    dispose?: boolean; closeThisWindow?: boolean; winRef?: Window | null;
  }): Promise<boolean>;
}
```

Components that have unsaved data register a `BeforeUnloadValidator` to prompt the user before the tab closes.

---

### `IWindowsManagerService` (`WINDOWS_MANAGER` token)
Wraps `window.WindowsManager` — manages child windows (pop-out viewers).

```typescript
interface IWindowsManagerService {
  HostWindow: WebClientWindow;
  GetRootWindowsManager(): IWindowsManagerService | undefined;
  CreateChildWindow(url: string, params?: Record<string, any>, config?: WindowConfig): Promise<WebClientWindow>;
}
```

---

### `IDataValidation` (`DATA_VALIDATION` token)
Wraps `window.DataValidation`.js — converts data from invariant (en-US) format to culture-specific display format (currency, dates, masked fields, etc.).

```typescript
interface IDataValidation {
  localizeData(
    data: string, dataType: DataValidationDataType,
    dataLength: number, mask: string, staticCharacters: string,
    full: boolean, bFormattedCurrency: boolean
  ): string | null;
  DataType: DataValidationDataType;
}
```

---

### `WebClientAppHostConfig`
Config bag injected by `AppHost.aspx` as `window.__WEB_CLIENT_APP_HOST_CONFIG__`:

```typescript
interface WebClientAppHostConfig {
  appContextGuid: string;
  dmsVirtualRoot: string;
  isDarkMode: boolean;
  isDarkModeAvailable: boolean;
  obToken: string;
  userName: string;
}
```

---

## 6. Enums Catalog

### `ViewerType`
Describes the rendering context in which the Angular UI is running:

| Value | Name | Description |
|-------|------|-------------|
| -1 | Unknown | Not determined |
| 0 | Unity | OnBase Unity Client |
| 1 | Web | OnBase Web Client |
| 2 | ObjectPop | Standalone browser pop-out |
| 3 | StandaloneAngular | Standalone Angular WorkView client |
| 4 | EmbeddedAngular | Embedded in Web Client, opens via WindowsManager |
| 5 | EmbeddedAngularEventPropogation | Embedded, propagates events to consumer (OBA stacked modals) |

---

### `WorkItemType`
Mirrors `Hyland.Workflow.WorkItemType` from the C# codebase:

`Unknown`, `All`, `None`, `Document`, `Folder`, `WorkViewItem`, `MedicalChart`, `EntityItem`, `EISMessageItem`, `ContentComposerBundle`, `Json`

---

## 7. TranslationPipe

```typescript
@Pipe({ name: 'obTranslate', standalone: true, pure: true })
export class TranslationPipe implements PipeTransform
```

Bridges the OnBase LRM to Angular templates:

```html
{{ 'DIALOG_TITLE' | obTranslate }}             <!-- simple lookup -->
{{ 'ITEMS_FOUND' | obTranslate:count }}        <!-- with args -->
```

**Mechanism**: Calls `window.Page.Get('ob_translations')` to retrieve the full dictionary, then calls `stringFormat(key, args)` from `flex-shared`. Falls back to the raw key if not found.

The `static getString()` method is also available for use in class code where the pipe is not accessible.

---

## 8. Mock Implementations

Every service interface has a matching mock class in `lib/shared/*/mock-*.ts`:

| Interface | Mock Class |
|-----------|-----------|
| `IOBStorageService` | `MockOBStorageService` — in-memory `_storage` object |
| `IPageService` | `MockPageService` — returns the variable name as-is |
| `ISessionCheckerService` | `MockSessionCheckerService` — no-op `logOut()` |
| `IUnloadManagerService` | `MockUnloadManagerService` — stores callbacks, `DeferredCleanup` returns `true` |
| `IWindowsManagerService` | `MockWindowsManagerService` — creates a fake div DOM node as the window |
| `WebClientWindow` | `MockWebClientWindow` — class implementing all window properties |

---

## 9. DI Token Map

```
  Token                    Interface                  Window global
  ────────────────────     ────────────────────────   ───────────────────────
  WEB_CLIENT_WINDOW        WebClientWindow            window (cast)
  WEB_CLIENT_APP_HOST_CONFIG  WebClientAppHostConfig  window.__WEB_CLIENT_APP_HOST_CONFIG__
  OB_STORAGE               IOBStorageService          window.OBStorage
  PAGE                     IPageService               window.Page
  SESSION_CHECKER          ISessionCheckerService     window.SessionChecker
  UNLOAD_MANAGER           IUnloadManagerService      window.UnloadManager
  WINDOWS_MANAGER          IWindowsManagerService     window.WindowsManager
  DATA_VALIDATION          IDataValidation            window.DataValidation
```

---

## 10. Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Isolate global state** | Every `window.*` global has a typed interface + DI token |
| **Zero production implementations** | This library only defines contracts; real implementations live in the Web Client JS files |
| **Full testability** | One mock class per interface; inject via providers in test beds |
| **String safety** | `TranslationPipe` uses `stringFormat` for typed arg interpolation |
| **Type-safe window** | `WebClientWindow` interface replaces all `(window as any)` casts |

---

*Generated from codebase analysis — `develop` branch, March 2026.*
