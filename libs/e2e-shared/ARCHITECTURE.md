# `e2e-shared` — Architecture Deep Dive

> **Audience**: QA engineers and developers authoring or maintaining Cypress E2E tests.  
> **Package**: `@hyland/e2e-shared`  
> **LOC**: 438 lines (TS src) · 0 specs · 0 HTML · 0 SCSS  
> **Usage score**: 97 import-lines (E2E only, no production code) · Tier 5 — Zero Prod Risk  
> **Companion document**: See [`libs/ARCHITECTURE.md`](../ARCHITECTURE.md) for the full cross-library picture.

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [The Big Picture](#2-the-big-picture)
3. [Public API](#3-public-api)
4. [Module Breakdown](#4-module-breakdown)
   - [Config Plugin (Node Process)](#41-config-plugin-node-process)
   - [Config Commands (Browser Process)](#42-config-commands-browser-process)
   - [Iframe Utilities](#43-iframe-utilities)
5. [Configuration Lifecycle](#5-configuration-lifecycle)
6. [Data Flow: Plugin ↔ Commands](#6-data-flow-plugin--commands)
7. [Usage Pattern in E2E Tests](#7-usage-pattern-in-e2e-tests)
8. [Key Interfaces](#8-key-interfaces)
9. [Design Principles](#9-design-principles)

---

## 1. Purpose

`e2e-shared` is a **Cypress utility library** shared across all E2E test projects (`apps/*-e2e`). It solves two recurring problems in E2E testing of OnBase applications:

1. **Configuration setup** — running expensive one-time server-side setup tasks (OnBase configuration, IIS resets) exactly once per test run, then sharing the resulting config data across all spec files.
2. **Iframe interaction** — providing a clean Cypress custom command for interacting with embedded iframes, which Cypress does not support out-of-the-box.

This library has **zero overlap with production code**. It imports only Cypress globals and contains no Angular modules.

---

## 2. The Big Picture

```
  Cypress Test Runner Process
  ┌─────────────────────────────────────────────────────────────────────┐
  │  NODE PROCESS (Cypress backend)                                     │
  │  ┌─────────────────────────────────────────────────────────────┐    │
  │  │  onBaseConfigPluginSetup(on)                                │    │
  │  │  • Registers task handlers: getConfig, storeConfig,         │    │
  │  │    isConfigurationDone, markConfigurationDone,              │    │
  │  │    clearConfigCache, log                                    │    │
  │  │  • Maintains configCache (in-memory, shared between specs)  │    │
  │  └─────────────────────────────────────────────────────────────┘    │
  │                       ▲  cy.task() calls                            │
  │  BROWSER PROCESS (Test spec / commands)                             │
  │  ┌─────────────────────────────────────────────────────────────┐    │
  │  │  register(modules)                                          │    │
  │  │  • cy.configure()  — runs config modules once per run       │    │
  │  │  • cy.getConfig<T>()  — retrieves stored config             │    │
  │  │  • cy.storeConfig()   — stores a config result              │    │
  │  │  • cy.iisReset()      — runs iisreset + waits               │    │
  │  │  • cy.pingIIS()       — pings the OnBase web server         │    │
  │  └─────────────────────────────────────────────────────────────┘    │
  │                                                                     │
  │  ┌───────────────────────┐                                          │
  │  │  registerSwitchToIframe()                                        │
  │  │  • cy.switchToIframe() — wraps iframe contents query             │
  │  └───────────────────────┘                                          │
  └─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Public API

```
  e2e-shared/src/index.ts
  ├── register(modules)                ← register config commands (support file)
  ├── ConfigurationModule<T>           ← interface for a config task module
  ├── onBaseConfigPluginSetup(on)      ← register plugin tasks (cypress config file)
  └── registerSwitchToIframe()         ← register cy.switchToIframe() command
```

> **Side-effect free**: none of these create commands or register anything at import time. All three registration functions must be explicitly called by the E2E project's support/config files.

---

## 4. Module Breakdown

### 4.1 Config Plugin (Node Process)

**File**: `lib/cypress/onbase-config/plugins/onbase-config-plugin.ts`  
**Function**: `onBaseConfigPluginSetup(on: Cypress.PluginEvents): void`

Called in each E2E project's `cypress.config.ts` setupNodeEvents:

```typescript
// apps/my-app-e2e/cypress.config.ts
export default defineConfig({
  e2e: {
    setupNodeEvents(on) {
      onBaseConfigPluginSetup(on);
    }
  }
});
```

**Registered tasks:**

| Task | Direction | Description |
|------|-----------|-------------|
| `getConfig(name)` | Node → Caller | Returns `{ config?, err? }` from cache by class name |
| `storeConfig(config)` | Caller → Node | Stores `{ name, config?, err? }` into cache |
| `isConfigurationDone()` | Node → Caller | Returns `Promise<boolean>` — was setup already run? |
| `markConfigurationDone()` | Caller → Node | Sets `configSetupFinished = true` |
| `clearConfigCache()` | Caller → Node | Resets cache + `configSetupFinished = false` |
| `log(msg)` | Caller → Node | `console.log` from the browser → node process |

The cache (`configCache`) is **in-memory** and lives for the entire duration of the Cypress runner process. This is the mechanism by which config data survives across different spec files.

---

### 4.2 Config Commands (Browser Process)

**File**: `lib/cypress/onbase-config/support/onbase-config-commands.ts`  
**Function**: `register<TConfig>(modules: ConfigurationModule<TConfig>[]): void`

Called in each E2E project's `support/e2e.ts`:

```typescript
// apps/my-app-e2e/src/support/e2e.ts
import { register, registerSwitchToIframe } from '@hyland/e2e-shared';
import { allConfigModules } from './configuration';

register(allConfigModules);
registerSwitchToIframe();
```

**Registered commands:**

| Command | Signature | Description |
|---------|-----------|-------------|
| `cy.configure()` | `(options?: ConfigureOptions) => void` | Runs all config modules once; skips if already done |
| `cy.getConfig<T>()` | `(configClass: new() => T) => Chainable<T>` | Retrieves a stored config by class name |
| `cy.storeConfig()` | `(config: CreatedConfiguration) => void` | Stores a config result in the node cache |
| `cy.iisReset()` | `() => void` | Navigates to blank page, runs `iisreset`, retries on failure |
| `cy.pingIIS()` | `(timeout: number) => void` | Posts a SOAP Ping to `Service.asmx` to warm up the app server |

---

### 4.3 Iframe Utilities

**File**: `lib/cypress/iframe/switch-to-iframe.ts`  
**Function**: `registerSwitchToIframe(): void`

Adds `cy.switchToIframe(callback)` using Cypress's `prevSubject` mechanism:

```typescript
cy.get('iframe#contentFrame').switchToIframe(() => {
  cy.get('[data-cy="submit"]').click();
});
```

This is necessary because Cypress does not natively support querying inside `<iframe>` elements. The command uses `cy.wrap(iframe.contents()).within(callback)` to scope subsequent queries inside the frame.

---

## 5. Configuration Lifecycle

```
  Test run starts (Cypress opens browser)
       │
       ▼
  Test spec's `before()` block calls cy.configure()
       │
       ├──► cy.task('isConfigurationDone') ──► Node: returns false (first run)
       │
       ├──► runConfiguration(modules)  [async]
       │         └── calls each module.func() in sequence
       │             (e.g. createDocument(), setPermissions(), etc.)
       │
       ├──► Each result → cy.storeConfig({ name, config })
       │         └──► cy.task('storeConfig') → Node: stores in configCache
       │
       ├──► cy.iisReset()   [resets IIS app pool]
       │
       ├──► cy.pingIIS()   [waits for server to return 200]
       │
       └──► cy.task('markConfigurationDone') → Node: configSetupFinished = true
  
  Next spec's `before()` block calls cy.configure()
       │
       └──► cy.task('isConfigurationDone') ──► Node: returns true → SKIPS
  
  Test retrieves config:
       cy.getConfig(MyConfigClass).then(config => { ... })
       └──► cy.task('getConfig', 'MyConfigClass') → Node: returns from cache
```

---

## 6. Data Flow: Plugin ↔ Commands

```
  Browser Process                    Node Process
  ─────────────────────              ─────────────────────
  cy.configure()
    │
    ├── cy.task('isConfigurationDone') ────────────────► configSetupFinished
    │   ◄──────────────────────────── boolean ──────────
    │
    ├── [if not done] runConfiguration(modules)
    │   (each module.func() called in browser)
    │
    ├── cy.task('storeConfig', {...}) ──────────────────► configCache[name] = ...
    │
    ├── cy.iisReset() → cy.exec('iisreset')
    │
    ├── cy.pingIIS() → cy.request(Service.asmx)
    │
    └── cy.task('markConfigurationDone') ──────────────► configSetupFinished = true
  
  cy.getConfig(MyClass)
    └── cy.task('getConfig', 'MyClass') ───────────────► return configCache['MyClass']
        ◄────────────────── { config?, err? } ─────────
```

---

## 7. Usage Pattern in E2E Tests

### Step 1: Support File Registration
```typescript
// apps/client-app-e2e/src/support/e2e.ts
import { register, registerSwitchToIframe } from '@hyland/e2e-shared';
import { configModules } from './app-configuration';

register(configModules);
registerSwitchToIframe();
```

### Step 2: Config Module Definition
```typescript
// apps/client-app-e2e/src/support/app-configuration.ts
export const configModules: ConfigurationModule<any>[] = [
  {
    returnType: DocumentConfig,
    func: () => createTestDocument()   // returns Promise<DocumentConfig>
  }
];
```

### Step 3: Use in Tests
```typescript
describe('Document Upload', () => {
  before(() => {
    cy.configure();   // runs once per Cypress session
  });

  it('uploads successfully', () => {
    cy.getConfig(DocumentConfig).then((config) => {
      cy.visit(`/document/${config.docId}`);
      cy.get('iframe').switchToIframe(() => {
        cy.get('[data-cy="upload"]').should('be.visible');
      });
    });
  });
});
```

---

## 8. Key Interfaces

```typescript
/** A configuration module: binding a config class to its async setup function */
interface ConfigurationModule<T extends object> {
  returnType: new () => T;   // class reference used as the cache key
  func: () => Promise<T>;    // async function that performs setup and returns config
}

/** Internal representation stored in cache */
interface CreatedConfiguration {
  name: string;     // class name (from configClass.constructor.name)
  config?: any;     // serializable config data
  err?: string;     // error message if setup failed
}

/** cy.configure() options */
interface ConfigureOptions<T> {
  configClass?: new () => T;    // run only this module (dev mode)
  cleanConfigCache?: boolean;   // force re-run (dev mode)
}
```

---

## 9. Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Run-once guarantee** | `isConfigurationDone` / `markConfigurationDone` pattern across all specs |
| **Cross-spec data sharing** | Node-process in-memory cache survives spec boundaries |
| **Type-safe retrieval** | `getConfig<T>(class)` uses class constructor as typed key |
| **Failure resilience** | Config errors stored as `err` string; IIS reset retried on failure |
| **Side-effect free imports** | All setup requires explicit function calls; no auto-registration |
| **Dev-friendly escape hatches** | `cleanConfigCache` option and single-spec `configClass` filter |

---

*Generated from codebase analysis — `develop` branch, March 2026.*
