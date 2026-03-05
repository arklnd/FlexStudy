# `flex-forms` — Architecture Deep Dive

> **Audience**: Engineers implementing form-based controls or integrating Angular Forms with the Flex runtime.  
> **Package**: `@hyland/flex-forms`  
> **LOC**: 1,507 lines (TS src) · 952 spec · 0 HTML · 0 SCSS  
> **Usage score**: 8 import-lines across the repo · Tier 4 — Low Effort  
> **Companion document**: See [`libs/ARCHITECTURE.md`](../ARCHITECTURE.md) for the full cross-library picture.

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [The Big Picture](#2-the-big-picture)
3. [Public API](#3-public-api)
4. [The Form Bridge Problem](#4-the-form-bridge-problem)
5. [BaseFormElement Interface](#5-baseformelement-interface)
6. [FormElementFacade (and Subtypes)](#6-formelementfacade-and-subtypes)
7. [NgFormsService](#7-ngformsservice)
8. [Validation System](#8-validation-system)
   - [Built-in Validation Types](#81-built-in-validation-types)
   - [ValidationType Contract](#82-validationtype-contract)
9. [FormSubmissionService](#9-formsubmissionservice)
10. [FormFieldErrorService](#10-formfielderrorservice)
11. [FormScreenBinding](#11-formscreenbinding)
12. [DefaultControlValueAccessorProxy](#12-defaultcontrolvalueaccessorproxy)
13. [Design Principles](#13-design-principles)

---

## 1. Purpose

`flex-forms` bridges Angular Reactive Forms with the Flex runtime. In the Flex architecture, forms are defined in configuration (`FormField` from `flex-config`), rendered in HTML templates, and driven by Web Component-style custom elements. Standard Angular Forms machinery does not automatically connect to these elements.

This library solves that: it provides:
1. The **`BaseFormElement` interface** — the contract a custom form element must implement
2. The **`FormElementFacade`** hierarchy — Angular-side wrappers around those elements  
3. The **`NgFormsService`** — programmatic creation and registration of `FormControl` instances from config
4. A **15-type validation system** — translates `FormFieldValidation` config into Angular `ValidatorFn` instances

---

## 2. The Big Picture

```
  flex-config
  ┌────────────────────────────┐
  │  FormField[]               │──── configuration input
  │  FormFieldValidation[]     │
  └────────────────────────────┘
               │
               ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │  flex-forms                                                      │
  │                                                                  │
  │  NgFormsService                                                  │
  │  • createControlForFieldAndBindToParentForm()                    │
  │    - finds parent <x-flex-form> element in DOM                   │
  │    - creates UntypedFormControl with validators from config      │
  │    - sets default value, restores pristine/dirty/touched state   │
  │    - calls form.registerControl(name, control)                   │
  │                                                                  │
  │  FormElementFacade (abstract)                                    │
  │  ├── FlexFormElementFacade  — wraps Flex-native form elements    │
  │  └── HabFormElementFacade  — wraps HAB form elements             │
  │                                                                  │
  │  Validation System                                               │
  │  ├── 15 ValidationType implementations                           │
  │  └── maps FormFieldValidation.type → Angular ValidatorFn         │
  │                                                                  │
  │  FormSubmissionService                                           │
  │  ├── subscribeToSubmissionEvent()                                │
  │  └── unsubscribeFromSubmissionEvent()                            │
  │                                                                  │
  │  FormFieldErrorService                                           │
  │  └── manages per-field error state and display                   │
  └──────────────────────────────────────────────────────────────────┘
               │
               ▼
  Angular Reactive Forms (FormGroup, FormControl, ValidatorFn)
```

---

## 3. Public API

```
  flex-forms/src/index.ts
  ├── FormElementFacade         — abstract base for form element wrappers
  ├── FormElementUtils          — static DOM traversal helpers
  ├── FormScreenBinding         — interface: binds a facade to its screen + subscription
  ├── DefaultControlValueAccessorProxy — ControlValueAccessor adapter
  ├── FormSubmissionService     — subscribe/unsubscribe to form submission events
  └── FormFieldErrorService     — query and display field-level validation errors
```

Note: `NgFormsService` is `providedIn: 'root'` but **not publicly exported**. It is used internally by `flex-runtime`.

---

## 4. The Form Bridge Problem

The Flex runtime renders UIs via HTML string templates with Web Component-style custom elements (`<x-flex-form>`, `<my-custom-input>`). Angular Forms does not automatically detect these elements as form controls.

The bridge works as follows:

```
  HTML template rendered by Flex runtime:
  ┌──────────────────────────────────────────────────────────────┐
  │  <x-flex-form flex-control-id="myForm">                      │
  │    <my-input   flex-control-id="name"   ...></my-input>      │
  │    <my-select  flex-control-id="type"   ...></my-select>     │
  │  </x-flex-form>                                              │
  └──────────────────────────────────────────────────────────────┘
               │
               ▼ flex-runtime calls NgFormsService
  For each form-field in FormField[]:
    1. Find the DOM element by flex-control-id
    2. FormElementUtils.findClosestFormElement(element) → gets <x-flex-form>
    3. If element.internalFormControl exists → use it (custom CVA elements)
       else → create new UntypedFormControl()
    4. Apply validators from FormFieldValidation[] config
    5. Set default value if present
    6. form.registerControl(fieldId, formControl) → adds to FormGroup
```

---

## 5. BaseFormElement Interface

The contract that any `<x-flex-form>` custom element must implement:

```typescript
interface BaseFormElement extends HTMLElement {
  isDirty: boolean;
  registerControl(name: string, control: AbstractControl): void;
  setValue(value: { [key: string]: any }, options?): void;
  patchValue(value: { [key: string]: any }, options?): void;
}
```

By extending `HTMLElement`, this interface allows Flex to find and interact with form elements via standard DOM queries while getting Angular Forms capabilities on top.

---

## 6. FormElementFacade (and Subtypes)

`FormElementFacade` is an **abstract adapter** that wraps a `BaseFormElement` and exposes it through a consistent Angular interface:

```typescript
abstract class FormElementFacade {
  get isDirty(): boolean
  registerControl(name: string, control: AbstractControl): void
  get submission(): Observable<any>   // fromEvent(element, 'submission')
  setValue(value, options?): void
  patchValue(value, options?): void
}
```

The `submission` observable listens to the native CustomEvent dispatched by `<x-flex-form>` when the form's submit button is clicked.

**Concrete subtypes**:

| Class | Purpose |
|-------|---------|
| `FlexFormElementFacade` | Wraps native Flex form elements (`<x-flex-form>`) |
| `HabFormElementFacade` | Wraps HAB (Hyland Application Builder) form elements with their specific event model |

The runtime creates the appropriate facade type based on which element selector was detected.

---

## 7. NgFormsService

`NgFormsService` is the **core form-setup coordinator** used by `flex-runtime` during screen initialization. It is `providedIn: 'root'`.

**Key method**: `createControlForFieldAndBindToParentForm()`

```
  Inputs:
    name             — the control field name
    element          — the DOM element with flex-control-id attribute
    field            — FormField from config (default value, validations)
    screenInstanceId — which screen instance this belongs to
    updateOn         — 'change' | 'blur' | 'submit' (default: 'change')
  
  Steps:
    1. Get controlId from element.getAttribute('flex-control-id')
    2. FormElementUtils.findClosestFormElement(element) → <x-flex-form>
    3. Log error if no parent form found
    4. If element.internalFormControl exists:
         a. Set default value preserving pristine/dirty/touched state
         b. Apply ValidatorFn[] from FormFieldValidation config
         c. Set updateOn strategy
    5. Register: form.registerControl(controlId, formControl)
```

The careful state restoration in step 4 (saving and restoring `pristine`, `dirty`, `touched`) ensures that setting a default value doesn't mark the form as dirty.

---

## 8. Validation System

### 8.1 Built-in Validation Types

15 validations are registered in `validationTypesList`:

| Validation Type | Angular Validator | Notes |
|----------------|-------------------|-------|
| `required` | `Validators.required` | Field must not be empty |
| `requiredNoWhitespace` | Custom | Required and must not be all whitespace |
| `min` | `Validators.min(n)` | Minimum numeric value |
| `max` | `Validators.max(n)` | Maximum numeric value |
| `minLength` | `Validators.minLength(n)` | Minimum string length |
| `maxLength` | `Validators.maxLength(n)` | Maximum string length |
| `pattern` | `Validators.pattern(re)` | Regex pattern match |
| `numeric` | Custom | Must be a valid number |
| `alphanumeric` | Custom | Must be alphanumeric |
| `requiredTruthy` | Custom | Value must be truthy (checkboxes) |
| `occursAfter` | Custom | Date must be after reference |
| `occursBefore` | Custom | Date must be before reference |
| `matDatepickerMax` | `MatDatepickerValidators.matDatepickerMax` | Material Datepicker upper bound |
| `matDatepickerMin` | `MatDatepickerValidators.matDatepickerMin` | Material Datepicker lower bound |
| `matDatepickerParse` | `MatDatepickerValidators.matDatepickerFilter` | Material Datepicker parse error |

### 8.2 ValidationType Contract

Each validation type implements the `ValidationType` interface:

```typescript
interface ValidationType {
  type: string;                        // matches FormFieldValidation.type
  createValidator(data?: any): ValidatorFn;  // produces the Angular ValidatorFn
  formatError?(error: any): string;    // formats the error for display
}
```

When `NgFormsService` processes a `FormField`, it iterates `FormFieldValidation[]`, looks up the matching `ValidationType` in `validationTypesList`, and calls `createValidator(validation.data)` to produce the Angular validator.

---

## 9. FormSubmissionService

Manages the lifecycle of form submission subscriptions:

```typescript
subscribeToSubmissionEvent(
  formScreenBinding: FormScreenBinding,
  onFormSubmission: (data: any) => void
): void

unsubscribeFromSubmissionEvent(formScreenBinding: FormScreenBinding): void
```

When a `<x-flex-form>` fires its `submission` CustomEvent, the facade's `submission$` Observable emits. `FormSubmissionService` connects this to the `flex-runtime`'s operation execution pipeline so that form submit triggers the configured `OperationList`.

---

## 10. FormFieldErrorService

Provides access to the current validation error state for a field:

- Tracks which form controls have been touched
- Returns the first active validation error for a field
- Used by form-element components to conditionally show error messages below inputs

---

## 11. FormScreenBinding

The `FormScreenBinding` interface is the glue object that ties a form facade to its screen context:

```typescript
interface FormScreenBinding {
  formElement: FormElementFacade;
  submissionSubscription?: Subscription;
}
```

Created by the runtime when a `<x-flex-form>` element is discovered on a screen. Stored and used by `FormSubmissionService` to manage subscriptions across screen navigation.

---

## 12. DefaultControlValueAccessorProxy

An implementation of Angular's `ControlValueAccessor` that acts as a default adapter for form-field elements that don't provide their own CVA:

```typescript
class DefaultControlValueAccessorProxy implements IControlValueAccessorProxy {
  // Standard CVA methods: writeValue, registerOnChange, registerOnTouched, setDisabledState
}
```

This allows the runtime to use standard Angular form binding mechanics even for elements that were not authored with Angular CVA in mind.

---

## 13. Design Principles

| Principle | Implementation |
|-----------|---------------|
| **DOM-agnostic interfaces** | `BaseFormElement extends HTMLElement` for native DOM interop |
| **Config-driven validation** | `FormFieldValidation[]` in config → `ValidatorFn[]` via registry |
| **State-safe defaults** | Default value application restores pristine/dirty/touched flags |
| **Observable submission** | Native CustomEvent → `fromEvent()` → RxJS pipeline |
| **Facade pattern** | `FormElementFacade` hierarchy decouples Angular code from custom element details |
| **Extensible validators** | `ValidationType` interface allows adding new validation types without modifying core |

---

*Generated from codebase analysis — `develop` branch, March 2026.*
