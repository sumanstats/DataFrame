# Testing, security, and reliability

## 1. Testing pyramid

### Core unit tests
- Constructors and schema validation.
- Type inference and explicit casts.
- Missing-value representation and predicates.
- Selection, filtering, mutation, sorting.
- Join key equality, duplicate expansion, suffixes, and ordering.
- Grouping/aggregation when implemented.
- Non-mutating behavior and input aliasing.

### Adapter tests
- Small golden fixtures for each format.
- Known input values and expected schema.
- Metadata preservation tests.
- Round-trip tests where writing is supported.
- Malformed and truncated files.
- Unicode, encodings, unusual names, blank cells, and missing values.

### Integration tests
- Raku NativeCall against the exact built native library.
- Native memory/resource cleanup.
- End-to-end read → transform → write → read.
- Browser viewer with hostile-looking text and large row counts.

### Property-based and stress tests
Where practical, test invariants such as:
- DataFrame row count matches every column length.
- Selecting columns does not alter row order.
- Filtering never increases row count.
- A left join retains all left rows at least once, while duplicate right keys may expand output.
- Writing and re-reading preserves documented values/types.
- Repeated malformed reads do not crash or leak resources.

## 2. Golden fixtures

Keep fixtures small, versioned, and legally distributable. Include provenance and expected results. Avoid committing confidential research data. Create synthetic fixtures for:
- Excel dates using both date systems.
- Shared and inline strings.
- Formula cells with cached values.
- CSV escaped quotes and embedded newlines.
- Statistical formats with value labels and user-defined missing values.
- RDS/RData with supported and unsupported object types.
- Encrypted XLSX variants explicitly claimed as supported.

## 3. Native code safety

- Pin or record native dependency versions and hashes where feasible.
- Prefer maintained releases with security fixes.
- Check all return codes and lengths.
- Validate sizes before allocation and arithmetic.
- Define buffer ownership and free each allocation exactly once.
- Use RAII inside C++ wrappers; never let exceptions cross C ABI boundaries.
- Do not trust file extensions or declared lengths.
- Add limits for file size, decompressed size, ZIP entries, string lengths, row count, column count, and nesting.
- Test repeated reads and failures under available leak/address sanitizers.
- Avoid writing temporary plaintext or decrypted files unless required; if needed, use restrictive permissions and reliable cleanup.

## 4. Password handling

- Never log passwords, derived keys, or decrypted workbook contents.
- Avoid storing passwords in object attributes after use.
- Do not put passwords on command lines or in URLs.
- Use a secure password input/callback mechanism appropriate to the environment.
- Clear sensitive buffers where feasible, while documenting the limitations of memory clearing in a managed runtime.
- Reject unsupported encryption schemes explicitly.
- Do not weaken verification or silently return corrupted output when a password is wrong.

## 5. ZIP/XML security

XLSX is an untrusted package input:
- Protect against ZIP bombs and extreme expansion ratios.
- Reject path traversal in archive member names.
- Restrict total decompressed size and number of entries.
- Avoid external entity resolution and network access during XML parsing.
- Validate worksheet relationships and cell coordinates.
- Handle malformed XML as a recoverable import error, not a process crash.

## 6. Browser security

- Escape cell values as text by default.
- Do not interpolate imported values into executable JavaScript or HTML.
- Use JSON serialization rather than manual string concatenation.
- Keep the server bound to loopback by default.
- Require explicit configuration to listen on a LAN/public interface.
- Do not expose arbitrary file reads or directory listings.
- Apply pagination and reasonable response-size limits.

## 7. Reproducibility

- Record Raku version, compiler/toolchain, native library versions, build flags, and OS in test reports.
- Pin dependencies or use reproducible package sources where practical.
- Do not fetch unreviewed binaries at runtime.
- CI should run the core test suite even when optional native adapters are unavailable, while clearly marking adapter tests as skipped rather than passed.

## 8. Error model

Use structured Raku exceptions or error types with:
- Operation/adapter name.
- File format and relevant path where safe.
- Stage of failure.
- Native error code/message if useful.
- Clear next action when possible.

Never expose raw pointers, crash details, secrets, or huge file contents in normal error messages.
