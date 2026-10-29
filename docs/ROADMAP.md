# Implementation roadmap

Do the project in small vertical slices. Every phase has a gate; do not start the next phase until the current acceptance criteria pass.

## Phase 0 — feasibility and API spike

Goals:
- Verify Raku availability/version in Replit.
- Verify C and C++ compilers, NativeCall, ZIP support, and dependency acquisition.
- Verify the project can also be built on the intended Windows environment.
- Prototype Raku syntax for method chaining, callable predicates, named derived columns, and error messages.
- Audit dependency licenses, upstream health, and build approaches.

Deliverables:
- A short tested environment report.
- A minimal API syntax decision.
- A dependency/version/licensing inventory.
- A go/no-go decision on Replit as the primary build environment.

Exit criteria: a tiny Raku package builds and runs on the selected target environment.

## Phase 1 — canonical data model

Implement only:
- DataFrame schema and ordered columns.
- Basic type descriptors.
- Dedicated missing-value semantics.
- Constructors from hash-of-arrays, list-of-hashes, and list-of-arrays.
- Shape validation and deterministic column ordering.
- Introspection: dimensions, names, types, preview.

Exit criteria: constructor and schema tests cover empty tables, zero columns, missing keys, unequal lengths, mixed types, and duplicate names.

## Phase 2 — core transformations

Implement:
- `select`, `filter`, `mutate`, `rename`, `arrange`.
- Non-mutating pipeline behavior.
- Explicit type conversion basics.
- Consistent error messages.

Exit criteria: transformations preserve schema and row-order rules, do not unexpectedly mutate source tables, and have tests for missing values and invalid column names.

## Phase 3 — joins and row operations

Implement:
- Inner and left joins first.
- Then right/full, semi, and anti joins.
- `distinct`, `slice`, row/column binding.
- Duplicate-key, suffix, and missing-key semantics.

Exit criteria: test unique, duplicated, missing, mixed-type, empty, and multi-key joins. Include many-to-many cases to verify expected row counts.

## Phase 4 — delimited text and FWF

Implement:
- CSV read/write through libcsv where appropriate.
- TSV.
- FWF.
- Type inference, explicit types, encoding, missing-token options.

Exit criteria: fixtures for quoting, embedded newlines, Unicode, empty fields, ragged records, large files, and malformed input.

## Phase 5 — Excel

Implement in this order:
1. `.xls` import through libxls.
2. Ordinary `.xlsx` read through ZIP + RapidXML package adapter.
3. `.xlsx` write through libxlsxwriter.
4. Encrypted `.xlsx` only after a separate feasibility and cryptography review.

Exit criteria: workbook fixtures include multiple sheets, sparse cells, shared strings, dates, booleans, formulas/cached values, blanks, unusual worksheet names, and malformed files. Encrypted fixtures must cover every explicitly claimed encryption scheme.

## Phase 6 — statistical formats and R data

Implement:
- ReadStat adapters one format at a time.
- librdata read and write for supported tabular objects.
- Metadata preservation and explicit unsupported-type warnings.

Exit criteria: fixtures from actual SAS/Stata/SPSS/R producers and round-trip tests where the library promises writing. Round-trip equivalence should compare values, types, missingness, and relevant metadata—not raw file bytes.

## Phase 7 — browser viewer

Implement optional local DataTables rendering:
- Bounded page serialization.
- Safe JSON and HTML escaping.
- Pagination, sorting/searching, column labels/types.
- Local-only server defaults and explicit network exposure.

Exit criteria: browser rendering handles Unicode and malicious-looking cell text safely, does not require sending all rows at once, and leaves the core package usable without a server.

## Phase 8 — stabilization

- Performance benchmarks.
- Memory and resource-limit testing.
- Fuzz/malformed-file testing where practical.
- Documentation and examples.
- API compatibility policy and release checklist.
- Windows/Replit/Linux build verification.

## Rules for each milestone

- Keep each task narrow enough to review.
- Write tests before adding another format.
- Do not replace a missing native dependency with a silent shell-out.
- Update the capability matrix and dependency inventory.
- Report tested platforms and commands honestly.
