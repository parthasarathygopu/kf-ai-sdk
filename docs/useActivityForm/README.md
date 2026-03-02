# useActivityForm

Form hook for workflow activities with automatic per-field sync and a single `handleSubmit` to complete the activity and advance the workflow.

## When to Use

**Use `useActivityForm` when:**

- Building a form for a workflow activity (employee leave request, manager approval, etc.)
- You need `handleSubmit` to complete the activity and advance the workflow
- Fields from prior activities should appear as readonly context

**Use something else when:**

- Building a BDO record form — use [`useBDOForm`](../useBDOForm/README.md) instead
- Listing activity instances in a table — use [`useActivityTable`](../useActivityTable/README.md) instead

## Imports

```tsx
import { useActivityForm } from "@ram_28/kf-ai-sdk/workflow";
```

## Quick Start

```tsx
import { useMemo } from "react";
import { useActivityForm } from "@ram_28/kf-ai-sdk/workflow";
import { EmployeeInputActivity } from "@/workflow/leave";

function LeaveRequestForm({ instanceId }: { instanceId: string }) {
  const activity = useMemo(() => new EmployeeInputActivity(), []);

  const { register, handleSubmit, errors, isLoading, isSubmitting } =
    useActivityForm(activity, { activity_instance_id: instanceId });

  if (isLoading) return <p>Loading...</p>;

  return (
    <form>
      <label>{activity.StartDate.label}</label>
      <input type="date" {...register(activity.StartDate.id)} />
      {errors.StartDate && <p>{errors.StartDate.message}</p>}

      <label>{activity.EndDate.label}</label>
      <input type="date" {...register(activity.EndDate.id)} />
      {errors.EndDate && <p>{errors.EndDate.message}</p>}

      <button type="button" disabled={isSubmitting}
        onClick={handleSubmit(() => {}, console.error)}>
        Submit Request
      </button>
    </form>
  );
}
```

## Usage Guide

### Registering Fields

Use the Activity instance's field to get the field ID, label, and metadata — never hardcode field names as strings.

```tsx
<label>{activity.StartDate.label} {activity.StartDate.required && <span>*</span>}</label>
<input type="date" {...register(activity.StartDate.id)} />
{errors.StartDate && <p>{errors.StartDate.message}</p>}
```

For **select, reference, boolean, and other custom components** that don't fire native change events, use `watch()` + `setValue()` instead of `register()` (see [Fields — Selection & Reference](../fields/README.md#selection--reference-fields) for full patterns):

```tsx
<Select
  value={watch(activity.LeaveType.id) ?? ""}
  onValueChange={(v) => setValue(activity.LeaveType.id, v)}
>
  <SelectTrigger>
    <SelectValue placeholder="Select leave type" />
  </SelectTrigger>
  <SelectContent>
    {activity.LeaveType.options.map((opt) => (
      <SelectItem key={opt.value} value={opt.value}>
        {opt.label}
      </SelectItem>
    ))}
  </SelectContent>
</Select>
```

Readonly fields are auto-disabled by `register()` — no manual disable needed.

### Context-Derived Readonly Fields

When a workflow has multiple activities, fields from prior activities appear as readonly context. The hook discovers them automatically from BP metadata, and `register()` returns `{ disabled: true }` for them.

In a manager approval form, the employee's StartDate, EndDate, LeaveType, and LeaveDays are readonly context, while ManagerApproved and ManagerReason are editable:

```tsx
import { useMemo } from "react";
import { useActivityForm } from "@ram_28/kf-ai-sdk/workflow";
import { ManagerApprovalActivity } from "@/workflow/leave";

function ManagerApprovalForm({ instanceId }: { instanceId: string }) {
  const activity = useMemo(() => new ManagerApprovalActivity(), []);

  const { register, handleSubmit, errors, isLoading, isSubmitting, watch, setValue } =
    useActivityForm(activity, { activity_instance_id: instanceId });

  if (isLoading) return <p>Loading...</p>;

  return (
    <div>
      {/* Readonly context from employee's activity — auto-disabled */}
      <label>{activity.StartDate.label}</label>
      <input type="date" {...register(activity.StartDate.id)} className="bg-gray-100" />

      <label>{activity.EndDate.label}</label>
      <input type="date" {...register(activity.EndDate.id)} className="bg-gray-100" />

      <label>{activity.LeaveDays.label}</label>
      <input type="number" {...register(activity.LeaveDays.id)} className="bg-gray-100" />

      {/* Editable manager decision fields */}
      <div>
        <input
          type="checkbox"
          checked={watch(activity.ManagerApproved.id) ?? false}
          onChange={(e) => setValue(activity.ManagerApproved.id, e.target.checked)}
        />
        <label>
          {activity.ManagerApproved.label}
          {activity.ManagerApproved.required && <span> *</span>}
        </label>
        {errors.ManagerApproved && <p>{errors.ManagerApproved.message}</p>}
      </div>

      <label>{activity.ManagerReason.label}</label>
      <textarea {...register(activity.ManagerReason.id)} rows={3} />
      {errors.ManagerReason && <p>{errors.ManagerReason.message}</p>}

      <button type="button" disabled={isSubmitting}
        onClick={handleSubmit(() => {}, console.error)}>
        Complete Review
      </button>
    </div>
  );
}
```

### Validation

Validation happens automatically. The hook validates field types, constraints (required, length, integerPart/fractionPart), and backend expression rules — all without any manual configuration.

Errors appear in `errors.FieldName.message`:

```tsx
{errors.StartDate && <p>{errors.StartDate.message}</p>}
```

### Handling Submit

Per-field sync automatically saves changes on blur/change via `activity.update()`, so there's no need for a manual save button. When the user is ready, `handleSubmit` validates the form, sends any remaining dirty fields, then completes the activity to advance the workflow.

`handleSubmit` is curried: `(onSuccess?, onError?) => (event?) => Promise<void>`.

- **`onSuccess(result)`** — Called with `CreateUpdateResponseType` (`{ _id: string }`) after the activity is completed.
- **`onError(error)`** — Called with `FieldErrors` (validation failure) or `Error` (API failure).

```tsx
<button type="button" disabled={isSubmitting}
  onClick={handleSubmit(() => { toast.success("Submitted"); onClose(); }, console.error)}>
  Submit Request
</button>
```

To validate before showing a confirmation dialog, use `trigger()`:

```tsx
const handleSubmitWithConfirm = async () => {
  const valid = await trigger();
  if (!valid) return;
  // Show confirmation UI, then invoke:
  handleSubmit(onSuccess, onError)(); // note the double invocation — curried
};
```

> **Don't** call `activity.update()` or `activity.complete()` manually — `handleSubmit` does it for you.

### Working with `item`

`item` represents the current activity instance. Each field on `item` is an accessor with methods to read, write, and validate that field's value.

**Instance-level members:**

```tsx
const { item } = useActivityForm(activity, { activity_instance_id: instanceId });

item._id;              // current activity instance ID
item.toJSON();         // all form values as plain object
await item.validate(); // trigger validation for all fields
```

**Field-level accessors** (`item.FieldName`):

```tsx
// Read/write
item.StartDate.get();                // current value
item.LeaveDays.getOrDefault(0);      // value or fallback
item.StartDate.set("2026-03-01");    // update value

// Validate
item.StartDate.validate();           // { valid: boolean, errors: string[] }

// Metadata
item.StartDate.label;                // display label
item.StartDate.required;             // is required?
item.StartDate.readOnly;             // is readonly?
```

**When to use `item` vs `register`/`watch`/`setValue`:**

- Use `register()` for standard HTML inputs (text, number, date)
- Use `watch()` + `setValue()` for custom components (select, checkbox, reference)
- Use `item.Field.get()` / `item.Field.set()` for programmatic read/write (event handlers, computed logic)

## Further Reading

- [API Reference](./api_reference.md) — All options, return values, and type definitions
- [Fields](../fields/README.md) — All 13 field classes, constraint getters, and attachment methods
- [Submit Leave Request](../examples/workflow/submit-leave-request.md) — Employee form submission
- [Approve Leave Request](../examples/workflow/approve-leave-request.md) — Manager approval with readonly context
- [Start New Workflow](../examples/workflow/start-new-workflow.md) — `workflow.start()` flow
- [Primitive Fields](../examples/fields/primitive-fields.md) — Form with String, Number, Boolean, Date, DateTime, Text
- [Complex Fields](../examples/fields/complex-fields.md) — Form with Select, Reference, User, File, Image

## Common Mistakes

- **Don't forget `useMemo` on the Activity.** `new ActivityClass()` must be wrapped in `useMemo(() => ..., [])`. Re-creating the instance on every render breaks the hook.
- **Don't create a manual save button.** Per-field sync handles auto-saving on blur/change. `handleSubmit` is for completing the activity, not saving drafts.
- **Don't set date `defaultValues` to empty string.** Use `undefined` instead. Empty strings cause type validation errors for Date and DateTime fields.
- **Don't use `register()` for select, checkbox, or reference components.** They don't fire native change events. Use `watch()` + `setValue()`.
- **Don't call `activity.update()` or `activity.complete()` manually.** `handleSubmit` handles the API calls. Calling them yourself will double-submit.
- **Don't render the form before `isLoading` is false.** The activity data and metadata are being fetched. Guard with `if (isLoading) return <Loading />`.
- **Don't forget `activity_instance_id` is required.** This is the activity instance ID (from `workflow.start()` or `getInProgressList()`), not a BDO record ID.
- **Don't pass a single options object.** `useActivityForm` takes 2 arguments: `useActivityForm(activity, { activity_instance_id })`. This is different from `useActivityTable` which takes a single object `useActivityTable({ activity, status })`. Passing `useActivityForm({ activity, activity_instance_id })` causes a type error.
