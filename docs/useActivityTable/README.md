# useActivityTable

Activity-integrated table hook for listing workflow instances with search, sorting, filtering, and pagination.

## When to Use

**Use `useActivityTable` when:**

- Displaying in-progress or completed workflow activity instances
- You need search, sort, filter, and/or pagination on activity data

**Use something else when:**

- Displaying BDO records (not workflow) — use [`useBDOTable`](../useBDOTable/README.md) instead
- Building a form for an activity instance — use [`useActivityForm`](../useActivityForm/README.md) instead

## Imports

```tsx
import { useActivityTable, ActivityTableStatus } from "@ram_28/kf-ai-sdk/workflow";
```

## Quick Start

```tsx
import { useMemo } from "react";
import { useActivityTable, ActivityTableStatus } from "@ram_28/kf-ai-sdk/workflow";
import { EmployeeInputActivity } from "@/workflow/SimpleLeaveProcess";

function PendingLeaveRequests() {
  const activity = useMemo(() => new EmployeeInputActivity(), []);

  const table = useActivityTable({
    activity,
    status: ActivityTableStatus.InProgress,
  });

  if (table.isLoading) return <p>Loading...</p>;
  if (table.error) return <p>Error: {table.error.message}</p>;

  return (
    <div>
      <ul>
        {table.rows.map((row) => (
          <li key={row._id}>
            {row.LeaveType.get()} — {row.StartDate.get()} to {row.EndDate.get()}
          </li>
        ))}
      </ul>

      <button
        onClick={table.pagination.goToPrevious}
        disabled={!table.pagination.canGoPrevious}
      >
        Previous
      </button>
      <span>
        Page {table.pagination.pageNo} of {table.pagination.totalPages}
      </span>
      <button
        onClick={table.pagination.goToNext}
        disabled={!table.pagination.canGoNext}
      >
        Next
      </button>
    </div>
  );
}
```

## Usage Guide

### Status Filter

Every `useActivityTable` call requires a `status` option that determines which items are fetched:

```tsx
const [status, setStatus] = useState(ActivityTableStatus.InProgress);

const table = useActivityTable({
  activity,
  status,
});
```

- **`ActivityTableStatus.InProgress`** — fetches the in-progress items of the current activity
- **`ActivityTableStatus.Completed`** — fetches the completed items of the current activity
- Raw values are `'inprogress'` (all lowercase, no underscore) and `'completed'`
- Always use the `ActivityTableStatus` constants — never raw strings
- Changing status re-fetches automatically (status is part of the query key)

### Accessing Row Data

Rows are `ActivityInstanceType` proxies, not plain objects. Access field values through the `.get()` method on each field accessor:

```tsx
table.rows.map((row) => (
  <tr key={row._id}>
    <td>{row.LeaveType.get()}</td>
    <td>{row.StartDate.get()}</td>
    <td>{row.EndDate.get()}</td>
    <td>{row.LeaveDays.get()}</td>
    <td>{row.Status.get()}</td>
  </tr>
));
```

- `row._id` is a direct string (no `.get()` needed)
- `row.LeaveType.get()` returns the field's current value
- Activity system fields: `row.BPInstanceId.get()`, `row.Status.get()`, `row.AssignedTo.get()`, `row.CompletedAt.get()`
- `row.toJSON()` converts the entire row to a plain object
- Editable fields also have `.set()`, but this is typically not used in a table context
- Persistence methods available on each row: `row.update()`, `row.save()`, `row.complete()`, `row.progress()`

### Search

Search filters results by matching a query string against a specific field. The input value updates immediately for a responsive UI, while the data fetch is debounced.

```tsx
<input
  type="text"
  placeholder={`Search by ${activity.EmployeeName.label}...`}
  value={table.search.query}
  onChange={(e) => table.search.set(activity.EmployeeName.id, e.target.value)}
/>
{table.search.query && (
  <button onClick={table.search.clear}>Clear</button>
)}
{table.isFetching && <span>Searching...</span>}
```

- **`search.set(field, query)`** — set the field and query text. The data fetch is debounced by 300ms. Pagination resets to page 1.
- **`search.clear()`** — clear the search field and query. Pagination resets to page 1.
- **`search.query`** — current query string (updates immediately, API call debounced)
- **`search.field`** — field currently being searched, or `null`
- Queries longer than 255 characters are silently ignored.

### Sorting

Sorting is controlled through the `sort` object. Column headers typically use `toggle()` for click-to-sort behavior:

```tsx
<th
  onClick={() => table.sort.toggle(activity.StartDate.id)}
  style={{ cursor: "pointer" }}
>
  {activity.StartDate.label}
  {table.sort.field === activity.StartDate.id &&
    (table.sort.direction === "ASC" ? " ↑" : " ↓")}
</th>
```

- **`sort.toggle(field)`** — cycles through ASC, DESC, then cleared. Toggling a different field starts at ASC.
- **`sort.set(field, direction)`** — set the sort field and direction directly. Pass `null` for both to clear.
- **`sort.clear()`** — remove sorting entirely.
- **`sort.field`** / **`sort.direction`** — current sort state, or `null` when no sort is active.
- **`initialState.sort`** format is an array of single-key objects: `[{ fieldName: "ASC" }]`.

### Filtering

The table includes an integrated [`useFilter`](../useFilter/README.md) instance at `table.filter`. Use it to build filter conditions that narrow down the table results:

```tsx
import { ConditionOperator, RHSType } from "@ram_28/kf-ai-sdk/table";

// Filter by leave type
table.filter.addCondition({
  Operator: ConditionOperator.EQ,
  LHSField: activity.LeaveType.id,
  RHSValue: "PTO",
  RHSType: RHSType.Constant,
});

// Date range filter
table.filter.addCondition({
  Operator: ConditionOperator.GTE,
  LHSField: activity.StartDate.id,
  RHSValue: "2026-01-01",
  RHSType: RHSType.Constant,
});

// Clear all filters
table.filter.clearAllConditions();

// Check if any filters are active
if (table.filter.hasConditions) {
  // ...
}
```

- **`table.filter.addCondition(condition)`** — add a filter condition
- **`table.filter.clearAllConditions()`** — remove all filter conditions
- **`table.filter.hasConditions`** — `true` if any conditions are active
- Filter changes automatically reset pagination to page 1, preventing empty pages after narrowing results.
- See the [useFilter documentation](../useFilter/README.md) for the full filter API.

### Pagination

Pages are 1-indexed (they start at 1, not 0). The default page is 1 and the default page size is 10.

```tsx
<div>
  <button
    onClick={table.pagination.goToPrevious}
    disabled={!table.pagination.canGoPrevious}
  >
    Previous
  </button>
  <span>
    Page {table.pagination.pageNo} of {table.pagination.totalPages}
    {" "}({table.pagination.totalItems} total)
  </span>
  <button
    onClick={table.pagination.goToNext}
    disabled={!table.pagination.canGoNext}
  >
    Next
  </button>
</div>

<select
  value={table.pagination.pageSize}
  onChange={(e) => table.pagination.setPageSize(Number(e.target.value))}
>
  <option value={10}>10</option>
  <option value={25}>25</option>
  <option value={50}>50</option>
</select>
```

- **`pagination.goToNext()`** / **`pagination.goToPrevious()`** — navigate forward/backward. No-op at boundaries.
- **`pagination.goToPage(n)`** — jump to a specific page. Clamped to `[1, totalPages]`.
- **`pagination.canGoNext`** / **`pagination.canGoPrevious`** — whether navigation is possible.
- **`pagination.setPageSize(n)`** — change the page size. Resets to page 1.
- **`pagination.pageNo`** / **`pagination.pageSize`** / **`pagination.totalPages`** / **`pagination.totalItems`** — current pagination state.

### Refetching

Call `refetch()` to manually refresh the table data. This refetches both the list and the count:

```tsx
const handleComplete = async (row) => {
  await row.complete();
  table.refetch();
};
```

This is the standard pattern after any mutation (complete, update, save) that should be reflected in the table. Also use `table.refetch()` after closing a form dialog that modifies an activity instance.

## Further Reading

- [API Reference](./api_reference.md) — All options, return values, and type definitions
- [My Pending Requests](../examples/workflow/my-pending-requests.md) — In Progress / Completed tabs
- [Filtered Activity Table](../examples/workflow/filtered-activity-table.md) — Date range filter on activities

## Common Mistakes

- **Don't forget `useMemo` on the Activity instance.** `new ActivityClass()` must be wrapped in `useMemo(() => ..., [])`. Re-creating the instance on every render causes infinite refetching.
- **Don't access row fields directly.** `row.LeaveType` is an accessor object, not the value. Use `row.LeaveType.get()` to read the value.
- **Don't use raw strings for status.** Use `ActivityTableStatus.InProgress` and `ActivityTableStatus.Completed`, not `'inProgress'` or `'in_progress'`.
- **Don't use 0-indexed pagination.** Pages start at 1. Passing `pageNo: 0` will produce incorrect results.
- **Don't forget to guard on `isLoading` before rendering rows.** While loading, `rows` is an empty array. Guard with `if (table.isLoading) return <Loading />` to avoid rendering an empty table.
- **Don't hardcode field names as strings.** Use the activity field IDs (`activity.StartDate.id`) instead of raw strings (`"StartDate"`). This keeps your code type-safe and resilient to field renames.
- **Don't use `table.data`** — the property is `table.rows`, same as `useBDOTable`.
- **`initialState.sort` format is `[{ fieldName: "ASC" }]`** — NOT `{ field, direction }`. Write `sort: [{ StartDate: "DESC" }]`, not `sort: [{ field: "StartDate", direction: "desc" }]`.
