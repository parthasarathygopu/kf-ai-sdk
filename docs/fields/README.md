# Fields

Field classes are the typed metadata layer for BDO properties. 13 classes extend `BaseField<T>`, each storing raw backend meta and exposing getters + `validate()`.

## Imports

```typescript
import {
  StringField,
  NumberField,
  BooleanField,
  DateField,
  DateTimeField,
  TextField,
  SelectField,
  ReferenceField,
  UserField,
  FileField,
  ImageField,
  ArrayField,
  ObjectField,
} from "@ram_28/kf-ai-sdk/bdo";
import type {
  StringFieldType,
  NumberFieldType,
  BooleanFieldType,
  DateFieldType,
  DateTimeFieldType,
  TextFieldType,
  SelectFieldType,
  ReferenceFieldType,
  UserFieldType,
  FileFieldType,
  ImageFieldType,
  FileType,
  ArrayFieldType,
  ObjectFieldType,
} from "@ram_28/kf-ai-sdk/types";
```

## CRITICAL: `.get()` Returns `T | undefined`

**Every** field accessor's `.get()` method returns `T | undefined` — **never just `T`**. This means you MUST null-guard the return value before passing it to typed functions:

```typescript
// ❌ WRONG — TypeScript error: 'string | undefined' not assignable to 'string | number'
const rowDate = new Date(row.start_time.get());
const label = format(row.start_time.get(), "MMM d");
getStatusBadge(row.status.get());  // param expects string, gets string | undefined

// ✅ CORRECT — Guard undefined before use
const startVal = row.start_time.get();
if (!startVal) return null;  // or provide a fallback
const rowDate = new Date(startVal);        // Now guaranteed string
const label = format(startVal, "MMM d");   // Now guaranteed string
getStatusBadge(row.status.get() ?? "");    // Fallback to empty string
```

This applies to ALL field types: `StringField.get()` → `string | undefined`, `NumberField.get()` → `number | undefined`, `DateTimeField.get()` → `string | undefined`, etc.

## Common Mistakes (READ FIRST)

1. **`item.Title` is not the value** — Use `item.Title.get()` to read, `.set()` to write.
2. **`.get()` returns `T | undefined`** — Always null-guard before passing to `new Date()`, `format()`, template literals, or typed function parameters. See section above.
3. **Don't confuse `StringField` (class) with `StringFieldType` (type alias)** — Different modules.
4. **`fetchOptions()` requires a parent BDO** — Standalone fields will throw.
5. **SelectField meta `Type` is `"String"`** — `Constraint.Enum` differentiates it. Only `SelectField` has the `.options` getter. `StringField` (even with an enum constraint) does NOT have `.options` — you must hardcode the enum values from the BDO file.
6. **Always use pre-built components for File/Image** — `<FileUpload>`, `<ImageUpload>`, `<FilePreview>`, `<ImageThumbnail>`.
7. **`<ImageUpload>` / `<FileUpload>` ONLY work with `ImageField` / `FileField`** — Never use them for `TextField` or `StringField`, even if the field name contains "image" or "file". A `TextField` named `image_urls` is still a text field — render it with `<Textarea>` or `<Input>`, not `<ImageUpload>`. Determine the component from the **field class** in the BDO file, not the field name.
8. **`<ImageUpload>` / `<FileUpload>` `field` prop MUST be `item.Field`, NOT `bdo.Field`** — The `upload()` and `deleteAttachment()` methods live on the form item accessor (created by the Item proxy), not on the BDO field class. Passing `bdo.Field` causes silent upload failures.
9. **BooleanField: use `watch()` + `setValue()`, never `register()`** — `<Switch>` and `<Checkbox>` components don't fire native change events. Using `register()` means the value never updates on toggle.

## Quick Start

```typescript
// Declare on a BDO class
readonly Title = new StringField({
  _id: "Title", Name: "Title", Type: "String",
  Constraint: { Required: true, Length: 255 },
});

// Access metadata
bdo.Title.id;       // "Title"
bdo.Title.label;    // "Title"
bdo.Title.required; // true
bdo.Title.length;   // 255

// Read/write via Item proxy
const item = await bdo.get("abc123");
item.Title.get();      // "My Product"
item.Title.set("New"); // update

// Form binding
<input {...register(bdo.Title.id)} />
```

## BaseField Contract

All fields share these getters: `id`, `label`, `readOnly`, `required`, `defaultValue`, `primaryKey`, `meta`.

Abstract method: `validate(value: T | undefined): ValidationResultType` → `{ valid, errors }`.

## Quick Reference

| Class                  | Value Type         | Extra Getters                                     | Async            |
| ---------------------- | ------------------ | ------------------------------------------------- | ---------------- |
| `StringField`          | `string`           | `length`                                          | —                |
| `NumberField`          | `number`           | `integerPart`, `fractionPart`                     | —                |
| `BooleanField`         | `boolean`          | —                                                 | —                |
| `DateField`            | `YYYY-MM-DD`       | —                                                 | —                |
| `DateTimeField`        | ISO string         | `precision`                                       | —                |
| `TextField`            | `string`           | `format`                                          | —                |
| `SelectField<T>`       | `T`                | `options`                                         | `fetchOptions()` |
| `ReferenceField<TRef>` | `TRef`             | `referenceBdo`, `referenceFields`, `searchFields` | `fetchOptions()` |
| `UserField`            | `{ _id, _name }`   | `businessEntity`                                  | `fetchOptions()` |
| `FileField`            | `FileType[]`       | —                                                 | —                |
| `ImageField`           | `FileType \| null` | —                                                 | —                |
| `ArrayField<T>`        | `T[]`              | `elementType`                                     | —                |
| `ObjectField<T>`       | `T`                | `properties`                                      | —                |

## Form Binding Patterns

| Field Type | Pattern                                   | Key Detail                                                                     |
| ---------- | ----------------------------------------- | ------------------------------------------------------------------------------ |
| String     | `register()`                              | `maxLength={field.length}`                                                     |
| Number     | `register()` with `type="number"`         | `step` from `fractionPart` (e.g. `2` → `"0.01"`, none → `"1"`)                 |
| Boolean    | `watch()` + `setValue()`                  | Checkboxes don't fire native change — never use `register()`                   |
| Date       | `register()` with `type="date"`           | Strict `YYYY-MM-DD` — never default to `""`                                    |
| DateTime   | `register()` with `type="datetime-local"` | `step="0.001"` for `"Millisecond"` precision — never default to `""`           |
| Text       | `register()` with `<textarea>`            | Check `field.format` (`"Plain"` \| `"Markdown"`) for conditional rendering     |
| Select     | `watch()` + `setValue()`                  | Static `field.options` from `Constraint.Enum`, or dynamic via `fetchOptions()` |
| Reference  | `watch()` + `setValue()`                  | Stores **full object**, not just ID — use `<ReferenceSelect>` component        |
| User       | `watch()` + `setValue()`                  | `{ _id, _name }` shape — use `fetchOptions()` + dropdown                       |
| File       | Component only                            | `<FileUpload>` (edit) / `<FilePreview>` (read-only)                            |
| Image      | Component only                            | `<ImageUpload>` (edit) / `<ImageThumbnail>` (read-only)                        |

## fetchOptions() Pattern

SelectField, ReferenceField, and UserField share this pattern:

```tsx
const [dropdownOpen, setDropdownOpen] = useState(false);
const { data: options = [] } = useQuery({
  queryKey: ["options", bdo.meta._id, field.id, item._id],
  queryFn: () => field.fetchOptions(item._id!),
  enabled: dropdownOpen && !!item._id,
  staleTime: Infinity,
});
```

## UI Components

| Component           | Field     | Mode      | Import Path                            | Key Props                                         |
| ------------------- | --------- | --------- | -------------------------------------- | ------------------------------------------------- |
| `<ReferenceSelect>` | Reference | Edit      | `@/components/system/reference-select` | `bdoField`, `instanceId`, `value`, `onChange`     |
| `<FileUpload>`      | File      | Edit      | `@/components/system/file-upload`      | `field`, `value`, `boId`, `instanceId`, `fieldId` |
| `<ImageUpload>`     | Image     | Edit      | `@/components/system/image-upload`     | `field`, `value`, `boId`, `instanceId`, `fieldId` |
| `<FilePreview>`     | File      | Read-only | `@/components/system/file-preview`     | `boId`, `instanceId`, `fieldId`, `value`          |
| `<ImageThumbnail>`  | Image     | Read-only | `@/components/system/image-thumbnail`  | `boId`, `instanceId`, `fieldId`, `value`          |

## Further Reading

- [API Reference](./api_reference.md) — Type signatures for BaseField and all 13 classes
- [Primitive Fields](../examples/fields/primitive-fields.md) · [Complex Fields](../examples/fields/complex-fields.md)
- [BDO](../bdo/README.md) · [useBDOForm](../useBDOForm/README.md)
