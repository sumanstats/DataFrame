# API design

Validate syntax with a small Raku prototype before implementing the core.

## 1. Design goals

- Familiar to dplyr/pandas users.
- Idiomatic Raku rather than a literal port of R syntax.
- Readable method chains.
- Explicit options for ambiguous behavior.
- Errors that point to the operation and column involved.
- Consistent behavior regardless of input format.

## 2. Constructors

Proposed conceptual entry points:
- `DataFrame.new(...)` for a hash of arrays/lists.
- `DataFrame.from-rows(...)` for a list of hashes.
- `DataFrame.from-matrix(...)` or `DataFrame.from-rows-of-arrays(...)` for a list of arrays.
- `read-csv(...)`, `read-xlsx(...)`, `read-xls(...)`, `read-sas(...)`, etc.

Examples of intended input shapes (not implementation code):
- Column-oriented: `{ name => ['A', 'B'], age => [20, 30] }`
- Row-oriented: `[{ name => 'A', age => 20 }, { name => 'B', age => 70 }]`
- Array rows: `[['A', 20], ['B', 30]]` with supplied names `['name', 'age']`.

Missing hash keys in row-oriented construction become `Missing`, not an ordinary string or `NaN`. I am not sure but I think `Missing` values can be represented as `NA`.  All rows must have the same schema after unioning keys; define a deterministic column-order rule, preferably first-seen order.

## 3. Pipeline style

Preferred style:

```text
df
  .select(<id name age>)
  .filter({ $_<age> >= 18 })
  .mutate(adult => { $_<age> >= 18 })
  .arrange(<age>)
```

This illustrates the intended readability only. Confirm exact Raku syntax and callable semantics before freezing the API.

## 4. Core verbs

| Verb | Purpose | Important semantic decision |
|---|---|---|
| `select` | Choose and reorder columns | Unknown names should error by default |
| `filter` | Keep matching rows | Missing/unknown predicate results are dropped by default, configurable later |
| `filter_out` | Excludes rows based on conditions | Exclude specific rows that meet a condition |
| `mutate` | Add or replace columns | Existing columns are replaced only when named explicitly |
| `rename` | Rename columns | Reject collisions unless an explicit policy is provided |
| `arrange` | Sort rows | Document ascending/descending and missing placement |
| `distinct` | Remove duplicate rows/keys | Specify which duplicate survives |
| `slice` | Select rows by positions | Define indexing convention clearly with its variants `slice_head()`, `slice_tail()`, `slice_min()`, `slice_max()`, `slice_sample()` |
| `relocate`	| Change the order of columns	| Moving columns around without changing their data |
| `group-by` | Create grouped view/plan | Group order and missing group keys documented |
| `summarise` | Aggregate per group | Aggregation and missing-value rules explicit |
| `bind-rows` | Stack rows | Schema union and type promotion documented |
| `bind-cols` | Append columns | Require equal row counts unless an explicit alignment mode is requested |


Other verbs to implement like that of dplyr are: rename_with(), transmute(), pull(), ungroup(), reframe(), rowwise(), count(), tally(), add_count(), add_tally(), bind_rows(), bind_cols()

## 5. Joins

Provide:
- `inner-join`
- `left-join`
- `right-join`
- `full-join`
- `semi-join`
- `anti-join`

Conceptual usage:

```text
people.left-join(visits, on => 'person_id')
```

For different key names, support a clear mapping form, for example `on => { person_id => patient_id }`, if Raku syntax and ergonomics test well. Also allow multiple keys. Do not silently join every same-named column by default; explicit keys are safer and easier to review.

Define duplicate-key expansion, output column suffixes, missing-key equality, and output order before implementation.

## 6. Row context vs column context

The simplest robust first API passes each row to a callable block for `filter` and `mutate`. This is intuitive and easy to test but may be slower than vectorized column expressions. Keep the API design open to future column expressions, such as a `col('age')` expression object, so that vectorized operations can be added without changing DataFrame storage.

Do not promise that every operation is vectorized in the first release. Document the initial evaluation model.

## 7. Missing values

Expose a visible, documented `Missing` value and predicates such as `is-missing`. Do not overload `Nil`, `NaN`, empty string, and missing into one meaning. Reader-specific sentinels must be normalized at import boundaries.

## 8. Type conversion

Plan explicit operations such as `as-integer`, `as-number`, `as-string`, `as-date`, and `as-datetime`, with error/warning policies. Do not rely on silent implicit conversions for join keys or identifiers.

## 9. I/O API

Use predictable names and options, e.g.:
- `read-csv(path, ...)`
- `read-tsv(path, ...)`
- `read-fwf(path, widths => ..., names => ...)`
- `read-xls(path, ...)`
- `read-xlsx(path, ...)`
- `read-sav`, `read-dta`, `read-sas7bdat`, `read-rds`, `read-rdata` as supported.
- `write-csv`, `write-xlsx`, and format-specific writers where available.

A generic `read-data`/`write-data` dispatcher can be added, but explicit functions should remain available for predictable options and errors.

## 10. Naming

Use Raku's normal hyphenated identifiers where idiomatic. Do not force snake_case everywhere just to match pandas. Avoid overloaded names with unclear meanings. The package name and namespace should be chosen after checking the Raku ecosystem for collisions.
