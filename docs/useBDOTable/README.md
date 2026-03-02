# useBDOTable

BDO-integrated table hook with data fetching, search, sorting, filtering, and pagination.

## When to Use

**Use `useBDOTable` when:**

- Displaying a list of BDO records in a table
- You need search, sort, filter, and/or pagination

**Use something else when:**

- Displaying workflow/activity records — use [`useActivityTable`](../useActivityTable/README.md) instead
- Building a form — use [`useBDOForm`](../useBDOForm/README.md) instead

## Imports

```tsx
import { useBDOTable } from "@ram_28/kf-ai-sdk/table";
```

## Quick Start

```tsx
import { useMemo } from "react";
import { useBDOTable } from "@ram_28/kf-ai-sdk/table";
import { BuyerProduct } from "@/bdo/buyer/Product";

function ProductsPage() {
  const product = useMemo(() => new BuyerProduct(), []);

  const table = useBDOTable({
    bdo: product,
    initialState: {
      sort: [{ Title: "ASC" }],
      pagination: { pageNo: 1, pageSize: 10 },
    },
  });

  if (table.isLoading) return <p>Loading...</p>;
  if (table.error) return <p>Error: {table.error.message}</p>;

  return (
    <div>
      <ul>
        {table.rows.map((row) => (
          <li key={row._id}>
            {row.Title.get()} - ${row.Price.get()}
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

### Accessing Row Data

Rows are `ItemType` instances, not plain objects. Access field values through the `.get()` method on each field accessor:

```tsx
table.rows.map((row) => (
  <tr key={row._id}>
    <td>{row.Title.get()}</td>
    <td>{row.Category.get()}</td>
    <td>${row.Price.get()}</td>
  </tr>
));
```

- `row._id` is a direct string (no `.get()` needed)
- `row.Title.get()` returns the field's current value
- `row.toJSON()` converts the entire row to a plain object
- Editable rows also have `.set()`, but this is typically not used in a table context

### Search

Search filters results by matching a query string against a specific field. The input value updates immediately for a responsive UI, while the API call is debounced.

```tsx
<input
  type="text"
  placeholder={`Search by ${product.Title.label}...`}
  value={table.search.query}
  onChange={(e) => table.search.set(product.Title.id, e.target.value)}
/>
{table.search.query && (
  <button onClick={table.search.clear}>Clear</button>
)}
{table.isFetching && <span>Searching...</span>}
```

- **`search.set(field, query)`** -- set the field and query text. The API call is debounced by 300ms. Pagination resets to page 1.
- **`search.clear()`** -- clear the search field and query. Pagination resets to page 1.
- **`search.query`** -- current query string (updates immediately, API call debounced)
- **`search.field`** -- field currently being searched, or `null`
- Queries longer than 255 characters are silently ignored.

### Sorting

Sorting is controlled through the `sort` object. Column headers typically use `toggle()` for click-to-sort behavior:

```tsx
<th
  onClick={() => table.sort.toggle(product.Title.id)}
  style={{ cursor: "pointer" }}
>
  {product.Title.label}
  {table.sort.field === product.Title.id &&
    (table.sort.direction === "ASC" ? " ↑" : " ↓")}
</th>
```

- **`sort.toggle(field)`** -- cycles through ASC, DESC, then cleared. Toggling a different field starts at ASC.
- **`sort.set(field, direction)`** -- set the sort field and direction directly. Pass `null` for both to clear.
- **`sort.clear()`** -- remove sorting entirely.
- **`sort.field`** / **`sort.direction`** -- current sort state, or `null` when no sort is active.
- **`initialState.sort`** format is an array of single-key objects: `[{ fieldName: "ASC" }]`.

### Filtering

The table includes an integrated [`useFilter`](../useFilter/README.md) instance at `table.filter`. Use it to build filter conditions that narrow down the table results:

```tsx
import { ConditionOperator, RHSType } from "@ram_28/kf-ai-sdk/table";

// Exact match
table.filter.addCondition({
  Operator: ConditionOperator.EQ,
  LHSField: product.Category.id,
  RHSValue: "Electronics",
  RHSType: RHSType.Constant,
});

// Range filter
table.filter.addCondition({
  Operator: ConditionOperator.GTE,
  LHSField: product.Price.id,
  RHSValue: 100,
  RHSType: RHSType.Constant,
});

// Clear all filters
table.filter.clearAllConditions();

// Check if any filters are active
if (table.filter.hasConditions) {
  // ...
}
```

- **`table.filter.addCondition(condition)`** -- add a filter condition
- **`table.filter.clearAllConditions()`** -- remove all filter conditions
- **`table.filter.hasConditions`** -- `true` if any conditions are active
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

- **`pagination.goToNext()`** / **`pagination.goToPrevious()`** -- navigate forward/backward. No-op at boundaries.
- **`pagination.goToPage(n)`** -- jump to a specific page. Clamped to `[1, totalPages]`.
- **`pagination.canGoNext`** / **`pagination.canGoPrevious`** -- whether navigation is possible.
- **`pagination.setPageSize(n)`** -- change the page size. Resets to page 1.
- **`pagination.pageNo`** / **`pagination.pageSize`** / **`pagination.totalPages`** / **`pagination.totalItems`** -- current pagination state.

### Refetching

Call `refetch()` to manually refresh the table data. This refetches both the list and the count:

```tsx
const handleDelete = async (id: string) => {
  await product.delete(id);
  table.refetch();
};
```

This is the standard pattern after any mutation (create, update, delete) that should be reflected in the table.

## Further Reading

- [API Reference](./api_reference.md) -- All options, return values, and type definitions
- [Product Listing](../examples/bdo/product-listing.md) -- Table with search, sorting, pagination
- [Edit Product Dialog](../examples/bdo/edit-product-dialog.md) -- Table + edit dialog + refetch
- [Filtered Product Table](../examples/bdo/filtered-product-table.md) -- Category + price range filters

## Common Mistakes

- **Don't forget `useMemo` on the BDO.** `new BdoClass()` must be wrapped in `useMemo(() => ..., [])`. Re-creating the instance on every render causes infinite refetching.
- **Don't use `table.data`** — the property is `table.rows`. Other table libraries use `data`, but this SDK uses `rows`.
- **Don't use `table.setSearch()`** — the correct API is `table.search.set(field, query)` with two arguments: field ID and query string.
- **Don't access row fields directly.** `row.Title` is an accessor object, not the value. Use `row.Title.get()` to read the value.
- **Don't use 0-indexed pagination.** Pages start at 1. Passing `pageNo: 0` will produce incorrect results.
- **Don't forget to guard on `isLoading` before rendering rows.** While loading, `rows` is an empty array. Guard with `if (table.isLoading) return <Loading />` to avoid rendering an empty table.
- **Don't hardcode field names as strings.** Use the BDO field IDs (`product.Title.id`) instead of raw strings (`"Title"`). This keeps your code type-safe and resilient to field renames.
- **`initialState.sort` format is `[{ fieldName: "ASC" }]`** — a single-key object, NOT `{ field, direction }`. Other libraries use `{ field: "Title", direction: "asc" }` but this SDK uses `[{ Title: "ASC" }]`. Note: the direction is uppercase `"ASC"` or `"DESC"`.
  ```tsx
  // ❌ WRONG — { field, direction } format from other libraries
  initialState: { sort: [{ field: product.Title.id, direction: "desc" }] }
  // ✅ CORRECT — single-key object with uppercase direction
  initialState: { sort: [{ Title: "DESC" }] }
  ```
- **Don't pass `bdo.field` where `bdo.field.id` is expected.** `bdo.Title` is a `StringField` object, not a string. Use `bdo.Title.id` to get the field ID string. Passing the field object causes TS2322 (`StringField` not assignable to `string`).
