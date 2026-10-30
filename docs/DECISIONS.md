# Architecture decisions and open questions

This file records decisions that should be made deliberately rather than emerging accidentally during implementation.

## Proposed decisions

### D-001 — One canonical DataFrame model
**Status:** proposed.

All readers and constructors normalize into the same core schema/column representation. No adapter-specific table type leaks into manipulation APIs.

### D-002 — Raku-first manipulation engine
**Status:** proposed.

Implement initial operations in Raku. Use native libraries for file I/O where specified. Optimize only after benchmarks.

### D-003 — Non-mutating transformations by default
**Status:** proposed.

`select`, `filter`, `mutate`, joins, and other transformations return a new DataFrame. Explicit in-place operations may be added later if there is a demonstrated need.

### D-004 — Missing-value representation
**Status:** proposed

The DataFrame library must support missing observations across heterogeneous typed columns and multiple input formats, including CSV, TSV, Excel, SAS, Stata, SPSS, and R data files.

R's NA semantics provide a useful behavioral model.

1. Provide a dedicated public missing-value constant named `NA`.
2. Do not use Raku `Nil` as the canonical internal representation of missing observations.
3. Represent missingness internally using a validity bitmap associated with each typed column.
4. Preserve the underlying column type when observations are missing.
5. Distinguish missing values from NaN, positive infinity, negative infinity, empty strings, and NULL-like objects.
6. Implement consistent missing-value propagation for arithmetic and comparisons.
7. Provide explicit missing-value operations, including `is-na`, `drop-na`, `replace-na`, `complete-cases`, and `na-count`.
8. Ensure every file-format reader converts source-specific missing-value representations into the canonical representation.
9. Make import rules configurable, particularly for empty strings and textual missing-value tokens.
10. Document missing-value behavior for filtering, sorting, aggregation, joins, and export.

Use R's familiar missing-value behavior where appropriate, while retaining idiomatic Raku APIs.

- Missing observations do not change column types.
- Missing values survive round-trip file imports and exports where the target format supports them.
- NaN and infinity remain distinguishable from missing values.
- Arithmetic, comparison, filtering, and aggregation have documented and tested behavior.
- All file readers and writers use consistent missing-value semantics.

Adopt R-like NA semantics at the public API level and Arrow-like validity bitmaps internally.


### D-005 — Missing-value behavior: design decisions
**Status:** Proposed

#### Join behavior

- Missing join keys match other missing join keys by default.
- Provide an option equivalent to dplyr's `na_matches = "never"`.
- Detect or guard against unexpectedly large many-to-many joins involving missing keys.

#### Empty fields during import

- Follow base R behavior by default.
- Empty numeric, integer, logical, and complex fields are missing.
- Empty character fields remain empty strings.
- Support an option to interpret empty character fields as missing.
- Respect quoted empty strings where the source format distinguishes them.

#### Export behavior

- CSV and text exports use `"NA"` as the default missing-value token.
- Allow users to configure the exported missing-value representation.
- Binary and statistical formats should use their native missing-value representations wherever possible.
- Preserve typed missing values and source-specific missing-value metadata when supported.

#### User-defined missing-value tokens

- Support configurable missing-value tokens during import.
- Apply token configuration per import operation.
- Do not interpret arbitrary numeric values as missing unless explicitly configured.
- Document how quoted and unquoted missing-value tokens are interpreted.

#### General principle

All readers must convert source-specific missing representations into the canonical DataFrame missing-value representation. All writers must convert canonical missing values into appropriate target-format representations.

Some samples for implementation:

- `read-csv("data.csv",:missing-values<NA N/A>,:empty-as-na(False));`
- `write-csv($df1, "output.csv", :na("NA"));`
- `$df1.left-join($df2,:by<patient_id>,:na-matches<na>);`
- `read-csv("data.csv", :empty-as-na(True));`




### D-006— Callable blocks first; expression DSL later
**Status:** proposed.

Begin with a clear callable row-context API. Prototype syntax before freezing it. Keep room for vectorized column expressions later.

### D-007— Browser viewer is optional
**Status:** proposed.

DataTables is an optional presentation package. It is not part of the DataFrame core and is not required for local scripts.

### D-008— Direct XLS import
**Status:** proposed.

Read `.xls` into the canonical model directly. CSV conversion is a separate optional utility, not an internal mandatory intermediate.

### D-009— Encrypted XLSX is a gated feature
**Status:** proposed.

Do not claim support until the Office encryption container, key derivation, verification, and decryption paths are implemented and tested against known fixtures. tiny-AES-c alone is not sufficient.


### D-010— Additional DataFrame Semantics Decisions
**Status:** proposed.


#### 1. Missing-value predicates

- `is-na` must return true for canonical NA and numeric NaN, following R's `is.na()` behavior.
- `is-nan` must identify numeric NaN.
- NaN must remain distinguishable from canonical NA internally.
- Importing NaN must not automatically convert it into NA.

#### 2. Expression evaluation

- Implement centralized three-valued logical semantics: True, False, and Unknown.
- Arithmetic involving canonical NA propagates NA.
- Comparisons involving canonical NA produce Unknown.
- Boolean operations must follow three-valued logic.
- NaN must retain its numerical semantics.
- Filtering retains only rows whose predicate evaluates to True.
- Higher-level operations must use a shared expression evaluation layer rather than implementing independent missing-value rules. Every higher-level DataFrame operation should use this layer.

#### 3. Entirely missing columns

- Introduce an internal Unknown or unresolved column type.
- A column containing only NA and no declared schema must not be arbitrarily inferred as integer, floating point, or string.
- An explicit schema must determine the type of an entirely missing column. `my $df = DataFrame.new({patient => <P001 P002>, outcome => [NA, NA]},:schema{outcome => Int});` This is syntax to make you understand.
- When an Unknown column is combined with a compatible typed column, the known type should take precedence.
- Type resolution must be documented and tested.

#### 4. Column ordering

- Do not promise insertion order for ordinary Raku Hash inputs.
- Hash-of-arrays constructors must use deterministic alphabetical column ordering.
- List-of-hashes constructors must use a deterministic alphabetical union of column names unless explicit ordering is supplied.
- Ordered input forms and explicit column-order specifications must preserve their supplied order.
- Missing keys in list-of-hashes inputs must be represented as canonical NA.

#### 5. CSV and TSV missing-token semantics

- CSV and TSV readers must retain field-quoting information until missing-token recognition is complete.
- Unquoted configured missing-value tokens represent missing observations.
- Quoted tokens represent literal strings.
- Quoted empty strings must remain distinguishable from unquoted empty fields.
- Empty numeric fields are missing by default.
- Empty character fields remain empty strings by default.
- Import behavior must be configurable.
- Writers and readers must support reliable round trips between this library's own CSV/TSV implementations.
- Document compatibility limitations with third-party readers.

#### 6. Join relationship safeguards

- Support one-to-one, one-to-many, many-to-one, and many-to-many relationship declarations.
- Warn by default when an unexpected many-to-many relationship occurs in equality joins.
- Explicitly declared many-to-many relationships must not generate this warning.
- Declared relationship constraints must be validated and violations must raise errors.
- Provide a configurable maximum output-row limit.
- Estimate or calculate join output size before allocating large result structures where practical.
- Document NA matching behavior separately from NaN matching behavior.

#### 7. General implementation principles

- Keep these semantics in shared internal components.
- Do not duplicate missing-value, ordering, or join-validation rules across individual file readers or public operations.
- Add unit tests for each rule and integration tests covering combinations of these rules.
- Do not silently change these decisions during implementation.

### D-011— Additional Decisions: Logical Unknown and NaN Semantics
**Status:** proposed.

#### 1. Three-valued logical semantics

The DataFrame expression engine shall implement strong Kleene three-valued logic.

Logical values are `True`, `False`, and `Unknown`.

The implementation shall satisfy:

| Expression | Result |
|---|---|
| `False AND Unknown` | `False` |
| `True AND Unknown` | `Unknown` |
| `True OR Unknown` | `True` |
| `False OR Unknown` | `Unknown` |
| `NOT Unknown` | `Unknown` |
| `Unknown AND Unknown` | `Unknown` |
| `Unknown OR Unknown` | `Unknown` |

Implicit Boolean coercion of `Unknown` shall not silently convert it to `True` or `False`. The implementation shall define and test the behavior of Boolean-context and short-circuit operations in ordinary Raku callable syntax.

The expression engine shall expose consistent logical semantics to filtering, mutation, selection, and aggregation operations.

#### 2. Logical Unknown versus unresolved column type

Logical `Unknown` and internal `UnresolvedType` are separate concepts.

- `Unknown` represents an indeterminate logical result.
- `UnresolvedType` represents a column whose data type cannot be inferred from its observations or declared schema.
- `NA` represents a missing observation.
- `NaN` represents a numeric Not-a-Number value.

These concepts shall have separate internal representations and shall never be implicitly substituted for one another.

A column containing only `NA` values without an explicit schema may have `UnresolvedType`. This does not make its individual observations logical `Unknown` values.

#### 3. NaN behavior in missing-value helpers

`is-na` shall return true for both bitmap-marked `NA` observations and numeric `NaN` values. `is-nan` shall identify numeric NaN values only.

By default, the following helpers shall follow `is-na` semantics:

- `drop-na`: removes observations that are NA or NaN.
- `complete-cases`: considers rows containing either NA or NaN incomplete.
- `na-count`: counts both NA and NaN.
- `replace-na`: replaces both NA and NaN.

Where useful, these helpers shall offer an explicit option to exclude NaN from their operations and act only on bitmap-marked missing observations. The default behavior shall remain consistent across these helpers.

NaN shall remain distinct from NA in storage, import, export, and numerical operations unless the user explicitly requests conversion.

Implement these in first prototype phase, not leave them until the full DataFrame implementation:

- Verify all three-valued truth-table combinations.
- Verify Unknown is not silently coerced in Raku Boolean context.
- Verify callable syntax and expression-engine syntax return equivalent results.
- Verify [NA, NA] without a schema has UnresolvedType, not logical Unknown.
- Verify is-na(NaN) and is-missing(NaN) differ.
- Verify drop-na, complete-cases, na-count, and replace-na all treat NaN consistently by default.
- Verify their explicit NaN-exclusion option works.


## Open questions to resolve during Phase 0

1. **Replit suitability:** Can the selected Replit environment install/build Raku, C/C++, and all chosen native dependencies reliably? Is Windows a first-class build target from the start?
2. **Expression syntax:** Which exact Raku syntax is ergonomic and reliable for `filter` and `mutate`? Build a small proof of concept.
3. **Name policy:** Are duplicate column names rejected by default? Recommendation: reject by default; offer explicit repair/rename helpers.
4. **Type inference:** What are the deterministic inference rules, and how can users disable inference or specify types?

## Change policy

When an open question is resolved:
- Record the decision and rationale here.
- Update the affected architecture/API/format document.
- Add tests for behavior that affects public semantics.
- Do not silently change the behavior after the API becomes public.


# Decision: DataFrame Construction API and Architecture

**Status:** Accepted  
**Decision area:** Public API, construction pipeline, internal architecture, performance  
**Applies to:** All DataFrame constructors and data importers

## 1. Core architectural decision

The library shall provide multiple public entry points for constructing DataFrames from different input shapes.

All public constructors and file readers MUST converge on one shared internal construction pipeline and produce the same canonical DataFrame representation.

The library MUST NOT implement separate, independent DataFrame storage or type-inference systems for each constructor or file format.

The fundamental design principle is:

- Public entry points interpret different input shapes.
- Input adapters normalize those shapes.
- A shared DataFrame builder validates and constructs columns.
- A single canonical DataFrame representation stores the resulting data.

This architecture shall support future constructors, import formats, and storage optimizations without requiring changes to the public DataFrame representation.

## 2. Public construction API

The initial public API shall include the following entry points:

| Entry point | Input orientation | Purpose |
|---|---|---|
| `DataFrame.new` | Column-oriented | Construct from a hash of columns |
| `DataFrame.from-rows` | Record-oriented | Construct from a list of hashes |
| `DataFrame.from-rows-of-arrays` | Row-oriented positional | Construct from arrays representing rows |
| `DataFrame.from-matrix` | Matrix-oriented | Construct from matrix-like numerical data |
| `read-csv` | File-oriented | Import CSV |
| `read-xlsx` | File-oriented | Import XLSX |
| `read-xls` | File-oriented | Import legacy XLS |
| `read-sas` | File-oriented | Import SAS data |
| `read-stata` | File-oriented | Import Stata data |
| `read-spss` | File-oriented | Import SPSS data |

All these entry points MUST return the same public DataFrame type.

### 2.1 Column-oriented constructor: DataFrame.new

`DataFrame.new` is the primary constructor and the preferred way to construct a DataFrame directly.

It accepts a hash of column names and their corresponding values.

Example:

```raku
my $people = DataFrame.new(
    columns => {
        id   => [1, 2, 3],
        age  => [20, NA, 17],
        name => ['A', 'B', 'C'],
    }
);
```

The constructor shall follow these rules:

1. Each hash key represents a column name.
2. Each hash value represents the observations in that column.
3. All columns MUST have equal lengths.
4. Column types MUST be inferred independently.
5. Missing observations MUST use the library's canonical `NA` representation.
6. An all-NA column without an explicit schema MUST have `UnresolvedType`.
7. An explicitly supplied schema MUST take precedence over type inference.
8. The constructor MUST NOT silently discard, truncate, or pad columns with inconsistent lengths.

An empty column collection MUST produce a valid empty DataFrame.

#### Explicit schema

The constructor shall support explicit column types.

Example:

```raku
my $people = DataFrame.new(
    columns => {
        id      => [1, 2, 3],
        outcome => [NA, NA, NA],
    },
    schema => {
        id      => Int,
        outcome => Int,
    },
);
```

An explicit schema resolves the type of an entirely missing column.

#### Explicit column ordering

The constructor shall support an explicit column order.

Example:

```raku
my $people = DataFrame.new(
    columns => {
        name => ['A', 'B'],
        age  => [20, 30],
    },
    column-order => ['name', 'age'],
);
```

Column ordering rules:

- Explicit `column-order` MUST be preserved.
- If no explicit order is provided, ordinary unordered Hash inputs MUST use deterministic alphabetical ordering.
- The implementation MUST NOT depend on undocumented Hash iteration order.
- Ordered input representations, such as ordered lists of pairs, MAY preserve their supplied order when supported.
- Duplicate column names MUST be rejected.

The initial public API shall require the named `columns` argument. Avoid ambiguous positional constructor forms in the initial release.

### 2.2 Record-oriented constructor: DataFrame.from-rows

`DataFrame.from-rows` accepts a list of hashes, with each hash representing one observation or record.

Example:

```raku
my $people = DataFrame.from-rows([
    { name => 'A', age => 20 },
    { name => 'B', age => 70 },
    { name => 'C', age => 17 },
]);
```

The result is equivalent to the column-oriented DataFrame:

| name | age |
|---|---:|
| A | 20 |
| B | 70 |
| C | 17 |

#### Missing keys

The constructor MUST determine the union of keys across all records.

A missing key in an individual record MUST be represented by `NA` in the corresponding column.

Example:

```raku
my $people = DataFrame.from-rows([
    { name => 'A', age => 20 },
    { name => 'B' },
    { name => 'C', age => 17 },
]);
```

Expected result:

| name | age |
|---|---:|
| A | 20 |
| B | NA |
| C | 17 |

A missing key MUST NOT cause a shorter column or a row-length mismatch.

Type inference shall be performed independently for each resulting column, using its non-missing observations.

If all observations in a column are missing and no schema is supplied, its type shall be `UnresolvedType`.

#### Explicit column selection and ordering

The constructor shall support an explicit list of columns.

Example:

```raku
my $people = DataFrame.from-rows(
    [
        { name => 'A', age => 20 },
        { name => 'B', age => 30 },
    ],
    columns => ['name', 'age'],
);
```

The explicit list determines column selection and ordering.

Keys absent from a particular record become `NA`.

The implementation should provide a strict validation option for applications that require records to have identical key sets. The default behavior shall permit missing keys and fill them with `NA`.

### 2.3 Positional row constructor: DataFrame.from-rows-of-arrays

This constructor accepts an array or list of arrays, with each nested array representing one row.

Example:

```raku
my $people = DataFrame.from-rows-of-arrays(
    [
        ['A', 20],
        ['B', 30],
        ['C', 17],
    ],
    columns => ['name', 'age'],
);
```

Expected result:

| name | age |
|---|---:|
| A | 20 |
| B | 30 |
| C | 17 |

Validation rules:

1. Column names MUST be supplied in the initial API.
2. Every row MUST contain the same number of elements.
3. The number of column names MUST equal the number of elements in each row.
4. Duplicate column names MUST be rejected.
5. Missing observations are allowed.
6. Each resulting column is independently type-inferred.
7. Malformed rows MUST NOT be silently truncated or padded.

Example of invalid input:

```raku
my $invalid = DataFrame.from-rows-of-arrays(
    [
        ['A', 20],
        ['B', 30, 'extra'],
    ],
    columns => ['name', 'age'],
);
```

This MUST produce an informative validation error identifying the malformed row and the expected and actual number of elements.

Generated column names may be considered as a future optional feature, but are not required for the initial API.

### 2.4 Matrix constructor: DataFrame.from-matrix

`DataFrame.from-matrix` shall be a separate constructor for genuinely matrix-like inputs.

A matrix is conceptually different from a heterogeneous DataFrame:

- A matrix is rectangular.
- Matrix elements generally share a common type.
- Matrices are commonly used for numerical and scientific computation.
- DataFrame columns may have different types.

Example:

```raku
my $measurements = DataFrame.from-matrix(
    [
        [10, 20],
        [15, 25],
        [12, 22],
    ],
    columns => ['before', 'after'],
);
```

The constructor shall validate dimensionality and column names.

It MUST NOT force heterogeneous data through a homogeneous numerical matrix representation.

Do not implement all array-based construction by routing through `from-matrix`.

## 3. File readers

File readers shall be independent public entry points.

Examples:

```raku
my $df = read-csv('people.csv');

my $df = read-xlsx('people.xlsx');

my $df = read-xls('people.xls');

my $df = read-sas('study.sas7bdat');

my $df = read-stata('study.dta');

my $df = read-spss('study.sav');
```

Each reader may expose format-specific options.

Example:

```raku
my $df = read-csv(
    'people.csv',
    delimiter => ',',
    na-values => ['NA', 'N/A'],
    header => True,
);
```

The file readers MUST NOT independently implement their own DataFrame storage, missing-value semantics, or type-inference systems.

Each reader shall:

1. Parse the source format.
2. Convert parsed values into normalized column data and source metadata.
3. Pass the normalized data to the shared construction pipeline.
4. Return the canonical DataFrame representation.

Format-specific parsing and conversion logic must remain isolated from the core DataFrame implementation.

For example, CSV, XLSX, and SAS readers may have different parsing mechanisms, but they must agree on the meaning of `NA`, column types, column ordering, and the canonical representation.

The construction architecture MUST support future streaming and batch-based file import.

## 4. Shared internal construction pipeline

All public constructors and file readers MUST use one shared DataFrame construction pipeline.

Conceptual flow:

```text
Public constructors and file readers
    |
    v
Input adapters
    |
    v
Shared DataFrameBuilder
    |
    +-- Input validation
    |
    +-- Column name and order validation
    |
    +-- Schema processing
    |
    +-- Type inference
    |
    +-- Missing-value normalization
    |
    +-- ColumnBuilder
    |
    v
Canonical DataFrame representation
```

The input adapters shall handle differences in input orientation.

The shared builder shall handle validation, schema resolution, type inference, missing-value normalization, and final assembly.

The canonical DataFrame representation shall be independent of the original input format.

### 4.1 Internal components

The implementation should use the following conceptual components.

| Component | Responsibility |
|---|---|
| `InputAdapter` | Normalize different input orientations |
| `ColumnBuilder` | Build individual columns |
| `TypeInference` | Infer and validate column types |
| `MissingValue` | Handle `NA`, `NaN`, and related semantics |
| `Schema` | Manage declared column types and constraints |
| `ColumnStore` | Own canonical column data |
| `DataFrameBuilder` | Coordinate validation and final DataFrame assembly |

These are architectural responsibilities, not a requirement to create a separate module for every small class immediately.

Keep the implementation modular, but avoid unnecessary abstraction and excessive fragmentation.

### 4.2 Single canonical representation

All DataFrames MUST use the same internal representation regardless of how they were constructed.

The recommended conceptual representation is:

```text
DataFrame
 |
 +-- Ordered column names
 |
 +-- Column-name lookup structure
 |
 +-- Row count
 |
 +-- Columns
       |
       +-- Column 1
       |     +-- Type metadata
       |     +-- Values
       |     +-- Validity information
       |
       +-- Column 2
       |     +-- Type metadata
       |     +-- Values
       |     +-- Validity information
       |
       +-- Column 3
             +-- Type metadata
             +-- Values
             +-- Validity information
```

The DataFrame shall be fundamentally column-oriented.

Each column shall maintain:

- Its name.
- Its type metadata.
- Its length.
- Its values.
- Its missing-value validity information.

A validity bitmap or equivalent compact validity representation is recommended for missing observations.

The public `NA` value MUST remain logically separate from the physical storage representation of missingness.

The initial implementation may use a simpler Raku-native storage implementation. However, the abstraction should permit future optimized typed buffers and native storage backends without redesigning the public API.

The design may take inspiration from Apache Arrow's columnar memory model, but full Arrow format compatibility is not required.

## 5. Performance requirements

Construction performance must be considered from the beginning.

### 5.1 Column-oriented construction

`DataFrame.new` should construct columns directly.

It MUST NOT convert column-oriented input into rows and transpose it back into columns.

### 5.2 Record-oriented construction

`DataFrame.from-rows` must convert records into columns efficiently.

Recommended algorithm:

1. Determine the union of column names.
2. Determine column order.
3. Create one builder per column.
4. Iterate through records.
5. Append each record's values to the corresponding column builders.
6. Append `NA` for missing keys.
7. Finalize each column.
8. Assemble the canonical DataFrame.

Avoid repeatedly appending to a growing DataFrame or creating unnecessary intermediate matrices.

### 5.3 Type inference

Type inference must be performed efficiently.

The implementation should avoid rescanning entire columns repeatedly.

Column builders should maintain sufficient type information as observations are added.

The type-promotion policy must be explicit, deterministic, and independently testable.

If later values are incompatible with the currently inferred type, the implementation must follow the documented type-promotion rules or raise an informative error in strict mode.

### 5.4 Memory ownership and copying

Avoid unnecessary data copying.

Safe ownership semantics take priority over premature zero-copy optimization.

The implementation may eventually support an explicit copying policy, for example:

```raku
DataFrame.new(
    columns => $columns,
    copy => True,
);
```

A future `copy => False` option may permit reuse of compatible storage when safe.

Do not promise zero-copy construction for arbitrary Raku Arrays, Lists, or Sequences. Zero-copy is only valid when the input storage is compatible with the internal representation and ownership/lifetime requirements are satisfied.

### 5.5 Streaming and batch construction

The shared construction pipeline should eventually support incremental or batch-based input.

Conceptual workflow:

```text
Read first batch
    |
    v
Append to column builders
    |
    v
Read next batch
    |
    v
Append to column builders
    |
    v
Finalize columns
    |
    v
Construct DataFrame
```

This is especially important for large CSV, XLSX, SAS, Stata, and SPSS imports.

Do not require every reader to create a complete intermediate list of rows before DataFrame construction.

Streaming support may be implemented after the initial constructors, but the architecture must not prevent it.

## 6. Suggested module organization

The following is a suggested organization, not a rigid requirement:

```text
lib/
└── DataFrame/
    ├── DataFrame.rakumod
    ├── Column.rakumod
    ├── Schema.rakumod
    ├── TypeSystem.rakumod
    ├── MissingValue.rakumod
    │
    ├── Construction/
    │   ├── Builder.rakumod
    │   ├── ColumnBuilder.rakumod
    │   ├── TypeInference.rakumod
    │   ├── FromColumns.rakumod
    │   ├── FromRows.rakumod
    │   ├── FromRowArrays.rakumod
    │   └── FromMatrix.rakumod
    │
    ├── IO/
    │   ├── CSV.rakumod
    │   ├── XLSX.rakumod
    │   ├── XLS.rakumod
    │   ├── SAS.rakumod
    │   ├── Stata.rakumod
    │   └── SPSS.rakumod
    │
    └── Storage/
        ├── ColumnStore.rakumod
        └── ValidityBitmap.rakumod
```

The final module structure may evolve as implementation progresses.

Do not create empty modules merely to match this diagram. Introduce modules when there is a concrete implementation responsibility that justifies them.

## 7. Testing and acceptance criteria

Every constructor MUST have dedicated tests.

### 7.1 DataFrame.new tests

Test:

- Construction from multiple columns.
- Mixed column types.
- Missing observations.
- Entirely missing columns.
- Explicit schema.
- Explicit column ordering.
- Deterministic ordering without explicit order.
- Empty DataFrame construction.
- Unequal column lengths.
- Duplicate column names.
- Invalid column names or unsupported values.

### 7.2 DataFrame.from-rows tests

Test:

- Uniform record keys.
- Missing keys in some records.
- Different keys across records.
- Entirely missing columns.
- Explicit schema.
- Explicit column selection and ordering.
- Strict key validation.
- Empty record list.
- Invalid record values.

### 7.3 DataFrame.from-rows-of-arrays tests

Test:

- Valid rectangular input.
- Mixed column types.
- Missing observations.
- Incorrect row lengths.
- Incorrect number of column names.
- Duplicate column names.
- Empty input.
- Invalid values.

### 7.4 DataFrame.from-matrix tests

Test:

- Valid rectangular matrix.
- Numeric matrix input.
- Column names.
- Invalid dimensions.
- Incompatible values.
- Empty matrix input.
- Correct conversion to the canonical DataFrame representation.

### 7.5 Cross-constructor equivalence

A particularly important acceptance criterion is that equivalent inputs produce equivalent DataFrames.

For example, the following constructions should produce equivalent results.
These examples explicitly request the same column order. Without explicit ordering, hash-based constructors use alphabetical ordering, while positional-row constructors preserve the supplied column-name order.

```raku
my $a = DataFrame.new(
    columns => {
        name => ['A', 'B'],
        age  => [20, 30],
    },
    column-order => ['name', 'age'],
);

my $b = DataFrame.from-rows(
    [
        { name => 'A', age => 20 },
        { name => 'B', age => 30 },
    ],
    columns => ['name', 'age'],
);

my $c = DataFrame.from-rows-of-arrays(
    [
        ['A', 20],
        ['B', 30],
    ],
    columns => ['name', 'age'],
);
```

All three DataFrames must have equivalent:

- Column names and order.
- Row counts.
- Column types.
- Values.
- Missing-value semantics.
- Public behavior.

Test equality using the library's eventual DataFrame equality or comparison facilities, not merely by comparing printed output.

### 7.6 Performance benchmarks

Establish construction benchmarks for:

- 100,000 rows × 10 columns.
- 1,000,000 rows × 10 columns.
- 100,000 rows × 100 columns.

Benchmark:

- Column-oriented construction.
- Record-oriented construction.
- Positional row-oriented construction.
- Peak memory usage.
- Type-inference overhead.
- Missing-value overhead.

Use benchmarks to guide optimization. Do not introduce complicated native storage or zero-copy behavior without measurable justification.

## 8. Implementation order

Implement construction in the following sequence:

1. Establish the missing-value and type-system semantics.
2. Define the Column abstraction and validity representation.
3. Implement ColumnBuilder.
4. Implement the shared DataFrameBuilder.
5. Implement and test `DataFrame.new`.
6. Implement and test `DataFrame.from-rows`.
7. Implement and test `DataFrame.from-rows-of-arrays`.
8. Implement and test `DataFrame.from-matrix`.
9. Implement `read-csv` using the shared pipeline.
10. Add XLSX, XLS, SAS, Stata, and SPSS readers using the same pipeline.
11. Introduce batch/streaming construction and performance optimizations.

Do not implement all constructors simultaneously.

The shared builder and canonical representation must be established before adding the additional constructors.

## 9. Rules for AI coding agents

When implementing this decision:

- Treat this design as accepted and authoritative unless the project owner explicitly approves a change.
- Do not silently change public constructor names or argument conventions.
- Do not create separate DataFrame storage implementations for different constructors.
- Do not implement file readers as independent DataFrame construction systems.
- Do not force heterogeneous data through a homogeneous matrix representation.
- Do not silently discard, truncate, or pad malformed input.
- Do not assume Hash iteration order is stable or meaningful.
- Do not conflate `NA`, `NaN`, logical `Unknown`, and `UnresolvedType`.
- Do not introduce premature zero-copy or native-storage complexity.
- Do not implement every planned feature in one step.
- Keep tests and documentation synchronized with implementation.

If a technical obstacle requires changing an accepted decision, explain:

1. Which accepted decision is affected.
2. Why the current design is technically unsuitable.
3. What alternative is proposed.
4. The consequences for API compatibility, performance, and maintenance.

Wait for project-owner approval before changing the accepted design.

## 10. Definition of completion

This decision is considered implemented only when:

- The canonical DataFrame representation exists.
- The shared construction pipeline is implemented.
- `DataFrame.new` passes its acceptance tests.
- The constructor produces correctly typed columns and preserves missing-value semantics.
- Additional constructors can use the same builder without introducing a separate storage system.
- Documentation describes actual behavior rather than merely planned behavior.

A feature is not complete merely because its code has been written. Its tests must pass and its acceptance criteria must be satisfied.