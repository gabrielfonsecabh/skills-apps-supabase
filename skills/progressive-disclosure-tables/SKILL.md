---
name: progressive-disclosure-tables
description: Use when building a screen where users start from an aggregate and need to reach the underlying records, a report, a summary table, a dashboard figure someone will question, a reconciliation or approval queue. Also use when choosing between a dialog, a sheet, a drawer or a new route for detail, when a table screen has become slow or has an unmanageable number of filters, or when users ask "where does this number come from?". Triggers / gatilhos: drill-down, detalhamento, abrir o lançamento, ver detalhes, tabela densa, filtros, painel lateral, modal, relatório, conciliação, aprovação.
---

# Progressive Disclosure in Data Tables

## Overview

In an internal tool the user rarely wants a record. They want to **understand a
number**, and the only way to trust it is to open it, and open what is inside it,
without losing the place they started from.

**Core principle: drill-down is layered, not navigational. Each level opens on top
of the previous one, and closing returns the user exactly where they were, with
their filters, scroll position and expanded rows intact.**

The moment a drill-down becomes `navigate('/detail/123')`, the user loses their
context and the back button becomes a gamble. That is the difference between a
tool people audit with and a tool people export to Excel and stop trusting.

## When to Use

- A table whose rows aggregate something (totals by category, period, entity)
- Any figure a user will be asked to justify in a meeting
- Reconciliation, approval, exception and queue screens
- Screens where "which records make up this?" is the main question

**Not for:** simple CRUD lists where the row *is* the record, wizards, or
single-record forms.

## The Chain

Every drill-down in a mature internal tool has the same five links. Build them in
this order; each is useless without the one before.

```
1. Dense table        aggregate rows, filters visible, totals in the footer
2. Clickable row      the whole row is the target, not a tiny chevron
3. Detail layer       the records behind that row, in a panel over the table
4. Inline action      fix it here, reclassify, approve, split, comment
5. Export             at every level, exporting exactly what is on screen
```

**Link 4 is the one teams skip, and the one that decides adoption.** A user who
drills into a wrong number and cannot correct it there will leave the system,
correct it elsewhere, and come back distrusting the screen. Put the action where
the diagnosis happens.

**Link 5 is not a nicety.** Export is how a finance or ops user checks your
software against their own. Exporting the filtered, drilled state, not the whole
table, is what makes it a verification rather than a dump.

## Choosing the Container

Measured across a mature internal app: **dialog 81 screens, sheet 9, drawer 2,
collapsible 2**. The distribution is not an accident.

| Container | Use when | Why |
|---|---|---|
| **Dialog** | A focused decision on one record: confirm, reclassify, edit a field, split a value | Centred, modal, short-lived. The user's answer is one action, then back to work |
| **Sheet** (side panel) | Rich detail that needs room and comparison with the table behind it | Slides from the edge; the table stays visible, so context is preserved |
| **Drawer** (bottom) | Mobile equivalent of the sheet | Reaches the thumb; wrong on desktop |
| **Collapsible / expanded row** | One more level of the *same* shape (subtotals under a total) | Keeps hierarchy inline; no layer at all |
| **New route** | The detail is a destination in its own right, deep-linkable and shareable | Only when the user genuinely leaves the analysis |

The honest test: **if the user must compare the detail against the row it came
from, use a sheet.** If they must simply answer a question about it, use a dialog.

Nested layers are fine and normal, a sheet that opens a dialog is a common,
readable pattern. A dialog that opens a sheet is not.

## Filtering: The Line That Decides Performance

**Filters must travel to the database. Row identity must not.**

The failure that teaches this: a screen loaded every record so it could filter in
the browser. At 26,000 rows the request exceeded the statement timeout, the query
returned empty, and the screen rendered zeroes, with no error. Users saw a
budget of R$ 0 and believed it.

```ts
// ❌ pull everything, filter on the client
const all = await fetchAllRows();          // dies silently at scale
const shown = all.filter(matchesFilters);

// ✅ the filter is part of the query
const { data } = await supabase.rpc('fn_rows', {
  p_status: filters.status, p_from: filters.from, p_to: filters.to,
  p_page: page, p_size: pageSize,
});
```

Corollaries:

- **Aggregate in the database.** A screen that needs a total must ask for the
  total, not for the rows to add up. This is the single largest performance win
  available on this kind of screen.
- **Paginate server-side**, and always order by a unique key, pagination without
  a unique sort silently skips and duplicates rows.
- **A list of selected ids may stay on the client**; the filter that produced them
  must not.
- **Never render an empty result as zero.** Distinguish "no rows matched" from
  "the query failed" and say which. Silent zeroes destroy trust permanently.

## Filter Ergonomics

Three components carry the whole experience, and they must agree:

1. **Filter in the column header**, where the data is. A separate filter panel far
   from the column it controls is guessing.
2. **Active filters as removable chips** above the table, plus a *clear all*. A
   user who cannot see that a filter is applied will report the numbers as wrong , 
   and they will be right, for the view they think they are looking at.
3. **Searchable selects.** Past ~20 options a native select is unusable, and admin
   dimensions routinely have hundreds.

Filter state belongs in one object so it can be serialised, cleared as a unit,
and later moved into the URL:

```ts
type FilterValue =
  | { type: 'text';   value: string }
  | { type: 'select'; value: string[] }
  | { type: 'range';  value: { from?: string; to?: string } };

const [filters, setFilters] = useState<Record<string, FilterValue>>({});
```

## Detail Panel Contract

Whatever the container, the detail layer owes the user five things:

1. **Provenance**, which row it came from, restated in the header. The user has
   forgotten by the time it opens.
2. **The records**, a real table, with its own pagination if it can be long.
3. **The reconciliation**, the sum of what is shown against the aggregate that
   was clicked. If they disagree, say so loudly; a silent discrepancy is worse
   than an error.
4. **The action**, the correction the diagnosis implies, performed here.
5. **The exit**, closing returns to the exact prior state. Never remount the
   parent table on close.

Keep the panel a controlled component. It receives the row and callbacks; it does
not fetch the parent's data or own the parent's state. That is what makes it
testable in isolation, and these panels deserve tests, because they are where
mutations happen.

## Quick Reference

| Situation | Answer |
|---|---|
| Row aggregates records | Whole row clickable → detail layer |
| Detail must be compared with the table | Sheet |
| One focused decision | Dialog |
| Same shape, one level deeper | Expand the row |
| Shareable, deep-linkable | Route |
| Table feels slow | Aggregate and filter in the database |
| Result is empty | Say whether it is "no matches" or "query failed" |
| User asks "is this right?" | Export the drilled, filtered state |
| Filter applied but not visible | Chips + clear all |

## Common Mistakes

| Mistake | Consequence |
|---|---|
| `navigate()` for detail | Context lost; back button unreliable; users stop drilling |
| Fetch all rows, filter in the browser | Silent timeout at scale; screen shows zeroes |
| Empty result rendered as `0` | Users act on a number that does not exist |
| Read-only detail panel | User leaves to fix it elsewhere and stops trusting the screen |
| Export ignores active filters | The export contradicts the screen; the screen loses |
| Tiny chevron as the only hit target | Users never discover the drill-down |
| Detail panel fetches the parent's data | Untestable, and the parent remounts on close |
| Filters scattered across the page | Nobody can tell what is currently applied |

## Red Flags: Stop

- A detail view implemented as a route without a deep-linking requirement
- `.filter(` over a full dataset that came from the network
- A total computed in JavaScript from rows fetched for that purpose
- A drill-down panel with no action inside it
- An export that does not receive the current filters
- A screen that renders `0` on an error path
