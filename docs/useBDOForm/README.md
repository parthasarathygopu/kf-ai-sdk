# useBDOForm

BDO-integrated form hook that wraps React Hook Form with automatic schema fetching, validation, per-field sync, and API submission.

## When to Use

**Use `useBDOForm` when:**

- Building create or edit forms backed by a BDO (Business Data Object)
- You want automatic validation without manual rules
- You want `handleSubmit` to call the API automatically (no manual `bdo.create()` / `bdo.update()`)

**Use something else when:**

- Building a workflow/activity form — use [`useActivityForm`](../useActivityForm/README.md) instead
- Building a filter or search form — use [`useFilter`](../useFilter/README.md) instead

## Imports

```tsx
import { useBDOForm } from "@ram_28/kf-ai-sdk/form";
```

## Quick Start

```tsx
import { useMemo } from "react";
import { useBDOForm } from "@ram_28/kf-ai-sdk/form";
import { SellerProduct } from "@/bdo/seller/Product";

function CreateProductForm({ id }: { id?: string }) {
  const product = useMemo(() => new SellerProduct(), []);

  const { register, handleSubmit, errors, isLoading, isSubmitting } =
    useBDOForm({ bdo: product, recordId: id });

  if (isLoading) return <p>Loading...</p>;

  return (
    <form onSubmit={handleSubmit((data) => console.log("Saved", data._id))}>
      <label>{product.Title.label}</label>
      <input {...register(product.Title.id)} />
      {errors.Title && <p>{errors.Title.message}</p>}

      <label>{product.Price.label}</label>
      <input type="number" step="0.01" {...register(product.Price.id)} />
      {errors.Price && <p>{errors.Price.message}</p>}

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Saving..." : "Save"}
      </button>
    </form>
  );
}
```

## Usage Guide

### Create Mode

When no `recordId` is passed (or it's `undefined`), the hook enters create mode. A draft is allocated on mount to get an `_id`.

```tsx
useBDOForm({ bdo: product });
```

### Update Mode

When `recordId` is a string, the hook enters update mode. The record is fetched and the form is populated.

```tsx
useBDOForm({ bdo: product, recordId: id });
```

### Registering Fields

Use the BDO instance's field to get the field ID, label, and metadata — never hardcode field names as strings.

```tsx
<label>{product.Title.label} {product.Title.required && <span>*</span>}</label>
<input {...register(product.Title.id)} />
{errors.Title && <p>{errors.Title.message}</p>}
```

For **select, reference, boolean, and other custom components** that don't fire native change events, use `watch()` + `setValue()` instead of `register()` (see [Fields — Selection & Reference](../fields/README.md#selection--reference-fields) for full patterns):

```tsx
<Select
  value={watch(product.Category.id) ?? ""}
  onValueChange={(v) => setValue(product.Category.id, v)}
>
  ...
</Select>
```

Readonly fields are auto-disabled by `register()` — no manual disable needed.

### Validation

Validation happens automatically. The hook validates field types, constraints (required, length, integerPart/fractionPart), and backend expression rules — all without any manual configuration.

Errors appear in `errors.FieldName.message`:

```tsx
{errors.Title && <p>{errors.Title.message}</p>}
```

### Handling Submit

Pass `handleSubmit` to your form's `onSubmit`. It validates, filters the payload, calls the API, and invokes your callbacks:

```tsx
<form onSubmit={handleSubmit(onSuccess, onError)}>
```

- **`onSuccess(data)`** — Called with `{ _id: string }` after a successful create or update.
- **`onError(error)`** — Called with `FieldErrors` (validation failure) or `Error` (API failure).

> **Don't** call `bdo.create()` or `bdo.update()` manually — `handleSubmit` does it for you.

### Working with `item`

`item` represents the current record instance. Each field on `item` is an accessor with methods to read, write, and validate that field's value.

**Instance-level members:**

```tsx
const { item } = useBDOForm({ bdo: product });

item._id;              // current record ID
item.toJSON();         // all form values as plain object
await item.validate(); // trigger validation for all fields
```

**Field-level accessors** (`item.FieldName`):

```tsx
// Read/write
item.Title.get();                // current value
item.Price.getOrDefault(0);      // value or fallback
item.Title.set("New Title");     // update value

// Validate
item.Title.validate();           // { valid: boolean, errors: string[] }

// Metadata
item.Title.label;                // display label
item.Title.required;             // is required?
item.Title.readOnly;             // is readonly?
```

**When to use `item` vs `register`/`watch`/`setValue`:**

- Use `register()` for standard HTML inputs (text, number, date)
- Use `watch()` + `setValue()` for custom components (select, checkbox, reference)
- Use `item.Field.get()` / `item.Field.set()` for programmatic read/write (event handlers, computed logic)

## Further Reading

- [API Reference](./api_reference.md) — All options, return values, and type definitions
- [Fields](../fields/README.md) — All 13 field classes, constraint getters, and attachment methods
- [Create Product](../examples/bdo/create-product.md) — Create form with validation and select fields
- [Edit Product Dialog](../examples/bdo/edit-product-dialog.md) — Table + edit dialog + refetch
- [Supplier Dropdown](../examples/bdo/supplier-dropdown.md) — Lazy-loaded reference field options
- [Primitive Fields](../examples/fields/primitive-fields.md) — Form with String, Number, Boolean, Date, DateTime, Text
- [Complex Fields](../examples/fields/complex-fields.md) — Form with Select, Reference, User, File, Image

## Common Mistakes

- **Don't forget `useMemo` on the BDO.** `new BdoClass()` must be wrapped in `useMemo(() => ..., [])`. Re-creating the instance on every render breaks the hook.
- **Don't set date `defaultValues` to empty string.** Use `undefined` instead. Empty strings cause type validation errors for Date and DateTime fields.
- **Don't mix `register()` and `watch()`+`setValue()` for the same field.** Pick one approach per field. `register()` for native inputs, `watch()`+`setValue()` for custom components.
- **Don't use `register()` for select, checkbox, switch, or reference components.** They don't fire native change events. Use `watch()` + `setValue()`.
- **Don't call `bdo.create()` or `bdo.update()` manually.** `handleSubmit` handles the API call. Calling them yourself will double-submit.
- **`handleSubmit` is curried — pass it directly to `onSubmit`, don't call it inside a handler.** It takes `(onSuccess, onError)` and returns an event handler. Write `<form onSubmit={handleSubmit(onSuccess, onError)}>`. NEVER write `<form onSubmit={(e) => handleSubmit(data)}>` or `await handleSubmit(data)` — this passes the wrong argument and the API call never fires.
  ```tsx
  // ❌ WRONG — handleSubmit receives FormEvent, not form data. API never fires.
  const onSubmit = async (data) => { await handleSubmit(data); };
  <form onSubmit={onSubmit}>

  // ✅ CORRECT — handleSubmit wraps your callback, validates, calls API, then invokes onSuccess
  <form onSubmit={handleSubmit(
    (result) => { toast.success("Created"); navigate("/products"); },
    (error) => { toast.error("Failed"); }
  )}>
  ```
- **Don't render the form before `isLoading` is false.** In create mode, the draft is being allocated. In update mode, the record is being fetched. Guard with `if (isLoading) return <Loading />`.
- **There is no `operation` field.** Create vs. update mode is determined solely by `recordId`: absent = create, present = update. Never pass `operation: "create"` or `operation: "update"`.
- **Guard `loadError` in edit forms.** After the `isLoading` check, add `if (loadError) return <ErrorMessage />` before rendering the form. Without this, `item` may be undefined in update mode.
  ```tsx
  if (isLoading) return <Loading />;
  if (loadError) return <p>Error: {loadError.message}</p>;
  ```
- **Don't pass `operation` even though the SDK type supports it.** The SDK has `UseBDOFormAutoOptionsType` that auto-infers create/update from `recordId`. Passing `operation: "update"` requires `recordId: string` (not `string | undefined`), causing TS errors with `useParams()`. Omit `operation` entirely:
  ```tsx
  // ❌ WRONG — TS2345 when id is string | undefined from useParams
  useBDOForm({ bdo, recordId: id, operation: "update" })

  // ✅ CORRECT — auto mode handles string | undefined
  useBDOForm({ bdo, recordId: id })
  ```
