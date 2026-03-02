# BDO (Business Data Object)

Type-safe, role-based data access layer. Each BDO class extends `BaseBdo<TEntity, TEditable, TReadonly>` and exposes CRUD methods filtered by role permissions.

## Imports

```typescript
// Page component — import the generated role-specific BDO class
import { AdminProduct } from "../bdo/admin/Product";
```

```typescript
// BDO class definition — used by the js_sdk generator
import { BaseBdo, StringField, NumberField, BooleanField, DateField, SelectField, ReferenceField } from "@ram_28/kf-ai-sdk/bdo";
import type { StringFieldType, NumberFieldType, BooleanFieldType, DateFieldType, SelectFieldType, ReferenceFieldType, SystemFieldsType } from "@ram_28/kf-ai-sdk/types";
import type { ListOptionsType } from "@ram_28/kf-ai-sdk/api/types";
```

## Common Mistakes (READ FIRST)

1. **`create()` returns `ItemType`, not `{ _id }`** — It returns a proxy-wrapped item with field accessors, not `CreateUpdateResponseType`. Use `item._id` to get the ID.
2. **Use `.get()` to read field values** — `item.Title` returns a field accessor object, not the value. Use `item.Title.get()`.
3. **Field names use `_name` not `name`** — System fields like `_created_by` return `{ _id, _name }`. Access the display name with `item._created_by.get()?._name`.
4. **Memoize BDO instances in React** — Wrap in `useMemo` to prevent re-creation on every render:
   ```typescript
   const product = useMemo(() => new AdminProduct(), []);
   ```
5. **`Filter` must be `ConditionGroupType`** — Always wrap conditions in `{ Operator: "And", Condition: [...] }`. Never pass a flat condition:
   ```typescript
   // ❌ WRONG — flat ConditionType causes TS2322
   await product.list({ Filter: { Operator: "EQ", LHSField: "Category", RHSValue: "Electronics" } });
   // ✅ CORRECT — ConditionGroupType wrapper
   await product.list({ Filter: { Operator: "And", Condition: [
     { Operator: "EQ", LHSField: "Category", RHSValue: "Electronics", RHSType: "Constant" }
   ] } });
   ```
6. **Use `PageSize` and `Page`, not `Take` or `Limit`** — `ListOptionsType` uses `PageSize` (number of items per page) and `Page` (1-indexed page number).
7. **Never spread ItemType proxies.** `list()`, `get()`, and `create()` return proxied `ItemType` objects. Spreading `{ ...item, extraProp }` copies enumerable keys but destroys the proxy — `.get()` and `.set()` stop working. Instead, wrap in a container object:
   ```typescript
   // ❌ WRONG — proxy is destroyed, .get() fails at runtime
   const enriched = items.map(item => ({ ...item, extra: fetchedData }));
   enriched[0].Title.get(); // TypeError: get is not a function

   // ✅ CORRECT — proxy preserved in container
   const enriched = items.map(item => ({ entry: item, extra: fetchedData }));
   enriched[0].entry.Title.get(); // works
   ```

## Quick Start

```typescript
const product = new AdminProduct();

// Create — returns ItemType (proxy with field accessors)
const item = await product.create({ Title: "Widget", Price: 29.99 });
item._id;           // "abc123"
item.Title.get();   // "Widget"
item.Price.get();   // 29.99

// List
const items = await product.list({ Page: 1, PageSize: 10 });
items[0].Title.get();

// Get single item
const found = await product.get("abc123");
found.Title.get();  // "Widget"

// Update — returns { _id: string }
await product.update("abc123", { Price: 39.99 });

// Delete — returns { status: string }
await product.delete("abc123");
```

## Usage Guide

### CRUD Operations

```typescript
const product = new AdminProduct();

// Create — returns ItemType with field accessors
const item = await product.create({
  Title: "Widget",
  Price: 29.99,
  Category: "Electronics",
});
item._id; // direct string access

// Get single item
const item = await product.get("abc123");

// Update — returns { _id: string }
const { _id } = await product.update("abc123", { Price: 39.99 });

// Delete — returns { status: string }
const { status } = await product.delete("abc123");

// Count
const total = await product.count();
const filtered = await product.count({
  Filter: {
    Operator: "And",
    Condition: [
      { Operator: "EQ", LHSField: "Category", RHSValue: "Electronics", RHSType: "Constant" },
    ],
  },
});
```

### ItemType Field Accessors

Every `get()`, `list()`, and `create()` call returns `ItemType` — a proxy with typed field accessors.

```typescript
const item = await product.get("abc123");

// Read values
item._id;                    // string (direct, not an accessor)
item.Title.get();            // string | undefined
item.Price.getOrDefault(0);  // number (never undefined)

// Write values (editable fields only)
item.Title.set("New Title");

// Field metadata
item.Title.label;        // "Product Title"
item.Title.required;     // true
item.Title.readOnly;     // false
item.Title.defaultValue; // undefined
item.Title.meta;         // raw backend field meta

// Validation
item.Title.validate();   // { valid: boolean, errors: string[] }
item.validate();         // validates all fields, returns { valid, errors }

// Serialize
item.toJSON();           // { Title: "New Title", Price: 29.99, ... }
```

### List with Filter, Sort, and Pagination

```typescript
const items = await product.list({
  Filter: {
    Operator: "And",
    Condition: [
      { Operator: "EQ", LHSField: "Category", RHSValue: "Electronics", RHSType: "Constant" },
      { Operator: "GTE", LHSField: "Price", RHSValue: 10, RHSType: "Constant" },
    ],
  },
  Sort: [{ Price: "DESC" }],
  Page: 1,
  PageSize: 20,
});
```

### Metric (Aggregation)

```typescript
const result = await product.metric({
  GroupBy: ["Category"],
  Metric: [{ Field: "Price", Type: "Avg" }],
});
// result.Data = [{ Category: "Electronics", Price: 299.5 }, ...]
```

### Pivot (Cross-Tabulation)

```typescript
const result = await product.pivot({
  Row: ["Category"],
  Column: ["IsActive"],
  Metric: [{ Field: "_id", Type: "Count" }],
});
// result.Data = { RowHeader: [...], ColumnHeader: [...], Value: [[...]] }
```

## Further Reading

- [API Reference](./api_reference.md) — full method signatures, parameter types, and return types
