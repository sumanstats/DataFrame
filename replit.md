# Instructions for Replit Agent

## Mission

Help build a maintainable, production-minded Raku DataFrame library according to this blueprint. The intended product is a Raku-native tabular data library with a fluent API, native file-format adapters, and an optional DataTables browser viewer.

## Before changing files

1. Read `README.md` and every relevant document in `docs/`.
2. Confirm which milestone in `docs/ROADMAP.md` is authorized.
3. Summarize the planned change and its acceptance tests before a large or cross-cutting implementation.
4. If a requirement conflicts with a document, identify the conflict and ask rather than silently choosing.

## Scope control

- The current project phase is planning. Do not write implementation code until explicitly asked to start a milestone.
- Do not attempt all file formats in one task.
- Do not generate placeholder implementations that claim a format is supported when it is not.
- Keep a capability matrix for implemented, experimental, and unsupported behavior.
- Update the relevant Markdown documents when a decision changes.

## Technical rules

- Raku is the primary language for the public API and table-manipulation logic.
- Native dependencies must be isolated behind adapters and, where useful, a small stable C ABI wrapper.
- Keep native pointers, allocation details, and library-specific errors out of public DataFrame APIs.
- Keep browser/UI code optional and separate from the DataFrame core.
- Prefer explicit, documented behavior over clever syntax or implicit coercion.
- Transformations should return new DataFrames by default. Any by-reference mutation must be explicit.
- Missing values must be represented distinctly from ordinary values and floating-point NaN.
- Column names, row order, type inference, duplicate names, and join cardinality need documented semantics.
- Do not use `eval`, arbitrary shell commands, or unsafe deserialization to process user data.
- Never log passwords, decrypted workbook contents, or sensitive file paths unnecessarily.
- Validate file signatures and structure, enforce configurable size/resource limits, and handle malformed inputs without crashing.
- Never claim encrypted `.xlsx` support merely because tiny-AES-c can perform AES operations.
- Avoid unnecessary dependencies. Record the purpose and license of each third-party library.

## Development workflow

For each authorized milestone:

1. Make the smallest coherent change.
2. Add or update tests alongside the implementation.
3. Run the relevant tests and report the exact commands and results.
4. Explain any tests that could not run because the environment lacks Raku, a compiler, or native libraries.
5. Update documentation and the capability matrix.
6. Do not mark a feature complete based only on compilation; verify behavior with fixtures and edge cases.

## Build and platform expectations

The intended development target includes Windows, Linux and MacOS. The development environements should have a working gcc compiler, cmake and ninja. The embedded C libraries should have a fixed version specified to ensure reproducibilty. When embedded C libraries are updated, updating the version of those libraries should pull the updated codes. 

## Definition of done

A change is done only when the relevant acceptance criteria pass, errors are handled, documentation matches behavior, and the implementation does not violate the architecture in `docs/ARCHITECTURE.md`.
