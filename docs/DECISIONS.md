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

### D-004 — Missing is distinct from NaN
**Status:** proposed.

Use a dedicated missing value or validity mask. I think NA would be appropriate for that. NaN remains a floating-point value unless an explicit import/operation policy says otherwise.

### D-005 — Callable blocks first; expression DSL later
**Status:** proposed.

Begin with a clear callable row-context API. Prototype syntax before freezing it. Keep room for vectorized column expressions later.

### D-006 — Browser viewer is optional
**Status:** proposed.

DataTables is an optional presentation package. It is not part of the DataFrame core and is not required for local scripts.

### D-007 — Direct XLS import
**Status:** proposed.

Read `.xls` into the canonical model directly. CSV conversion is a separate optional utility, not an internal mandatory intermediate.

### D-008 — Encrypted XLSX is a gated feature
**Status:** proposed.

Do not claim support until the Office encryption container, key derivation, verification, and decryption paths are implemented and tested against known fixtures. tiny-AES-c alone is not sufficient.

## Open questions to resolve during Phase 0

1. **Replit suitability:** Can the selected Replit environment install/build Raku, C/C++, and all chosen native dependencies reliably? Is Windows a first-class build target from the start?
2. **Column storage:** Begin with generic Raku arrays or typed storage per column? Recommendation: start simple behind a Column interface and benchmark.
3. **Expression syntax:** Which exact Raku syntax is ergonomic and reliable for `filter` and `mutate`? Build a small proof of concept.
4. **Name policy:** Are duplicate column names rejected by default? Recommendation: reject by default; offer explicit repair/rename helpers.
5. **Type inference:** What are the deterministic inference rules, and how can users disable inference or specify types?
6. **Date/time:** Which date/time representation and timezone policy will be used internally?
7. **XLSX ZIP dependency:** Which maintained ZIP library has the best portable build and license fit?
8. **Office encryption:** Which specific Standard/Agile encryption variants are in scope? Is a maintained implementation available that can be safely integrated?
9. **ReadStat and librdata:** Which versions and platforms will be supported? What exact read/write formats and metadata are promised?
10. **Viewer server:** Should the optional viewer use a small Raku HTTP server dependency or emit a static HTML file? Recommendation: begin with static HTML for small data, then add a loopback server/paging if needed.
11. **Package namespace:** Check the Raku ecosystem for existing package names and conventions before choosing the distribution name.
12. **License policy:** Decide whether all dependencies must be permissively licensed or whether LGPL dependencies are acceptable with the required compliance.

## Change policy

When an open question is resolved:
- Record the decision and rationale here.
- Update the affected architecture/API/format document.
- Add tests for behavior that affects public semantics.
- Do not silently change the behavior after the API becomes public.
