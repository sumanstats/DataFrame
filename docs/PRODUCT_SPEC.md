# Product specification

## 1. Vision

Create an intuitive, robust DataFrame library for Raku. It should make common tabular operations discoverable and concise while preserving Raku's expressive programming model. It should read and write common spreadsheet, delimited-text, statistical, and R data formats through native libraries, and optionally render a DataFrame in a browser using DataTables.

## 2. Target users and use cases

- Raku programmers manipulating rectangular datasets.
- Researchers importing data from Excel and statistical packages.
- Users combining data from multiple sources and preparing analysis-ready tables.
- Users inspecting tables in a browser without moving manipulation logic into JavaScript.
- Developers who want an extensible API with predictable types and missing-value semantics.

## 3. Functional requirements

### DataFrame construction

Construct a DataFrame from:
- A hash of arrays/lists: each key is a column name.
- A list of hashes: each hash is a row; missing keys become the library's missing value.
- A list/array of lists/arrays, with explicit or generated column names.
- Explicit column objects or typed vectors in a future advanced API.

Validate rectangular shape. Define behavior for unequal column lengths, empty inputs, duplicate column names, and empty names. Prefer clear errors over silent truncation or recycling. Do not automatically recycle unequal columns unless a deliberate, documented compatibility option is added later.

### Manipulation

Required operations:
- Inspect: dimensions, column names, types, preview/head/tail, summary, row access.
- `select`: choose/reorder columns; later support rename and helper selectors.
- `filter`: keep rows according to predicates.
- `mutate`: create or replace derived columns.
- `arrange`: sort rows by one or more columns with explicit missing-value ordering.
- `rename`, `distinct`, `slice`, `bind-rows`, `bind-cols`.
- `group-by` and `summarise` in a later milestone.
- Joins: inner, left, right, full, semi, and anti.
- I/O: read/write supported formats through consistent entry points.
- Optional browser viewer using DataTables.

### Type system

At minimum plan for logical, integer, floating-point, string, date, datetime, and missing values. Preserve source metadata when possible. Factors/categorical variables, labelled values, time zones, decimal precision, and 64-bit integers need explicit policies. Avoid silently converting identifiers with leading zeros to numbers.

### Missing values

Use a dedicated missing sentinel or a value-plus-validity representation. Floating-point NaN remains a numeric value with distinct semantics unless the user opts into treating NaN as missing. Define:
- Predicate behavior for missing values.
- Filter behavior when a predicate is missing/unknown.
- Join matching for missing keys.
- Aggregation behavior with missing values.
- Export encoding per file format.

### I/O

Initial planned formats:
- Legacy Excel `.xls` via libxls.
- `.xlsx` via ZIP/package parsing plus RapidXML for XML parts.
- Password-encrypted `.xlsx`, after a dedicated encryption feasibility/security milestone.
- `.xlsx` writing via libxlsxwriter.
- CSV via `libcsv`.
- TSV.
- Fixed-width files (FWF).
- SAS, Stata, and SPSS via ReadStat.
- R `.rds` and `.rdata` via librdata, within the subset of R objects supported by that library.

“Read/write supported formats” means real, tested support for documented subsets—not just recognizing extensions. All these dependencies should be embedded within the module. 

### Browser view

A separate optional viewer renders tabular data with DataTables. The DataFrame core must remain usable in a terminal or script without a browser, HTTP server, or JavaScript runtime. Browser rendering is not a substitute for manipulation in Raku.

## 4. Non-functional requirements

- Correctness and predictable semantics are more important than early speed.
- Portable design across Replit/Linux and Windows, subject to verified native dependency support.
- Clear errors for malformed files and unsupported features.
- No unbounded memory or CPU consumption from untrusted files.
- Reproducible builds and documented native dependency versions.
- A strong test suite with fixtures for each supported format.
- API stability after initial public release.
- Good documentation with examples, migration notes, and capability matrix.

## 5. Explicit non-goals for the first release

- Reimplementing all of dplyr, pandas, or data.table.
- A full Excel formula calculation engine.
- Preserving every formatting feature of imported Excel workbooks.
- Arbitrary R object serialization beyond librdata's supported tabular/data-frame subset.
- Distributed execution, lazy query optimization, GPU support, and out-of-core storage in the first milestone.
- Making a web app mandatory for local use.

## 6. Product success criteria

A user can create a DataFrame, chain select/filter/mutate/arrange, join two frames, and export the result without knowing which file library produced the input. Missing-value behavior is consistent across operations. Every advertised format is covered by real fixtures and has documented limitations.
