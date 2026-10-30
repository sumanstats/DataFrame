# Architecture

## 1. Architectural overview

Use a layered architecture:

1. **Public Raku API** — DataFrame methods, constructors, expressions, errors, and documentation.
2. **Core table model** — ordered columns, row count, schema, types, and missing-value representation.
3. **Manipulation engine** — selection, filtering, mutation, sorting, joins, grouping, and aggregation.
4. **I/O facade** — format detection, options, dispatch, and consistent import/export errors.
5. **Format adapters** — libxls, XLSX package/XML reader, libcsv, ReadStat, librdata, libxlsxwriter, TSV/FWF.
6. **Native bridge layer** — Raku NativeCall declarations and small C/C++ wrappers where the raw ABI is awkward.
7. **Optional browser viewer** — a local HTTP serving layer and DataTables front end, isolated from core.

Data flows through a canonical internal model. Format adapters may preserve source metadata as optional column/table metadata, but the manipulation engine should not depend on the source format.

## 2. Core representation

Recommended first implementation: an ordered collection of named columns with a common row count. Each column has:
- A name.
- A logical type.
- A typed vector.
- Missing-value information.
- Optional metadata (labels, source type, format, timezone, etc.).

Do not prematurely implement a sophisticated packed columnar engine. Start with simple, correct Raku storage behind a stable Column interface, then optimize measured bottlenecks. Keep the public API independent of the storage implementation.

A column-oriented representation is appropriate because select, mutate, aggregation, and many vector operations work naturally on columns. Row-oriented views can be created for predicates and iteration without making rows the primary storage.

## 3. Schema and type inference

- Schema is explicit and inspectable.
- Constructors can infer types but must follow documented deterministic rules.
- Readers should use source metadata when reliable and retain identifiers as strings when source metadata says they are strings.
- Do not infer a whole column as numeric if that destroys leading zeros or loses nonnumeric values without warning.
- Provide an explicit type override/conversion API.
- Preserve distinction between integer, floating-point, logical, text, dates, datetimes, and missing values.
- Avoid implicit cross-type coercion in joins unless a documented, explicit coercion policy permits it.

## 4. Expression evaluation

Start with Raku callable blocks rather than attempting to clone R's non-standard evaluation or implement a full expression compiler.

Candidate initial form:
- Row predicate: `filter({ $_<age> >= 18 })`
- Derived column: `mutate(bmi => { $_<weight> / ($_<height> ** 2) })`

The exact syntax is provisional. Before committing, build a tiny API spike to check Raku parsing, block binding, method chaining, and useful error messages. Prefer a small, explicit row context initially. A later column-expression DSL can add vectorized arithmetic and lazy execution without breaking the core model.

Do not make API correctness depend on undocumented operator overloading or fragile parsing tricks.

## 5. Transformation semantics

Default to non-mutating transformations: each operation returns a new DataFrame. This is easier to reason about in pipelines and avoids surprising changes to original input objects. If performance later requires in-place operations, provide separately named explicit operations.

Operations must preserve row order unless their purpose is to reorder rows. `select` preserves row order; `filter` preserves the relative order of retained rows; joins document their output ordering and duplicate-key behavior.

## 6. Join engine

Implement joins against canonical column values and a shared key-comparison policy. Support one or multiple key columns. Specify:
- `by` as one name, multiple names, or a left/right key mapping.
- Suffixes for duplicate non-key column names.
- Duplicate key handling (many-to-many expansion is possible and must be documented).
- Missing-key matching policy.
- Key type compatibility and coercion.
- Stable output order.
- Semi/anti join output schema.
- Empty input and zero-row behavior.

Do not rely on a hash-map key representation that conflates missing values, numeric representations, or different types.

## 7. I/O architecture

Expose a consistent `read-*` / `write-*` facade and format-specific options. Use extension as a hint, not the sole trust mechanism; validate file signatures and package structure. Keep format adapters independently testable.

Adapters should return a normalized import result containing table data, schema, metadata, and warnings. Errors should identify the format, stage, and source location when possible, without exposing sensitive data.

Prefer direct import to the canonical model. CSV can be an optional export target for legacy `.xls` conversion, but should not be a mandatory intermediate for DataFrame import because it can discard source types and metadata.

## 8. Native boundary

Use Raku NativeCall for a stable C ABI. For C++ dependencies such as RapidXML, provide a narrow `extern "C"` wrapper rather than exposing C++ templates, exceptions, or object layouts directly to Raku.

For every native boundary:
- Define ownership and lifetime of buffers.
- Copy data into Raku-owned storage unless zero-copy semantics are fully justified and documented.
- Translate native error codes into Raku exceptions.
- Never allow C++ exceptions to cross a C ABI.
- Ensure cleanup on all success and failure paths.
- Test repeated reads/writes for leaks and crashes.

## 9. Browser viewer

The viewer is an optional package/module:
- Serialize a bounded page of values to safe JSON.
- Render using DataTables.
- Escape text and avoid injecting imported cell contents as HTML.
- Do not expose arbitrary filesystem paths or bind to a public network interface by default.
- Avoid sending the full table to the browser for very large datasets; use paging.
- Document local-only defaults and explicit network exposure options.

The viewer must not be imported automatically by the core DataFrame module.

## 10. Performance strategy

First optimize correctness, then benchmark:
1. Constructor and column access.
2. Select/filter/mutate.
3. Joins with unique and duplicate keys.
4. CSV and statistical-file ingestion.
5. Large table memory use.
6. Browser pagination.

Keep benchmarks separate from correctness tests. Avoid speculative SIMD, native compute kernels, or lazy query plans until profiling identifies real bottlenecks.
