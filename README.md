# Raku DataFrame — project blueprint

A planned Raku-native DataFrame library for intuitive tabular data manipulation, broad statistical-file interoperability, and optional browser rendering.

## Product direction

Build one coherent DataFrame API inspired by the best parts of `dplyr` and pandas, expressed naturally in Raku. File readers and writers are adapters around a common typed table model. The core manipulation engine must not depend on Excel, CSV, SAS, Stata, SPSS, R serialization, or the browser.

## Read these documents first

1. [`replit.md`](replit.md) — persistent instructions for Replit Agent.
2. [`docs/PRODUCT_SPEC.md`](docs/PRODUCT_SPEC.md) — requirements and scope.
3. [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — components and data flow.
4. [`docs/API_DESIGN.md`](docs/API_DESIGN.md) — proposed public API.
5. [`docs/FILE_FORMATS.md`](docs/FILE_FORMATS.md) — formats, libraries, and important caveats.
6. [`docs/ROADMAP.md`](docs/ROADMAP.md) — phased implementation plan.
7. [`docs/TESTING_SECURITY.md`](docs/TESTING_SECURITY.md) — validation, security, and compatibility.
8. [`docs/DECISIONS.md`](docs/DECISIONS.md) — architectural decisions and unresolved questions.

## Non-negotiable principles

- Do not begin implementation until the architecture and first milestone are agreed.
- Do not implement every file format at once. Implement each format step by step. Ask before implementing, what you are planning and what are its caveats.
- Do not make the DataFrame depend on native-library pointers or browser code.
- Do not use `NaN` as the universal missing-value marker.
- Do not claim encrypted XLSX support from AES primitives alone.
- Do not shell out to R, Python, LibreOffice, or Excel as the default implementation strategy.
- Every feature must have tests and documented semantics.
- Prefer clear, maintainable Raku over premature optimization.

## Initial acceptance target

The first vertical slice should create DataFrames from Raku data structures, inspect and select columns, filter rows, create derived columns, perform one join, and test missing-value behavior. It should work before any file-format adapter or browser viewer is added.

## Reference links

- Raku: https://raku.org/
- NativeCall: https://docs.raku.org/language/nativecall
- Replit Agent documentation: https://docs.replit.com/replitai/assistant/
- Replit project skills: https://docs.replit.com/replitai/skills
