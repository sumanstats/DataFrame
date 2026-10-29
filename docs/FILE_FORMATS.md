# File-format and native-library plan

## General rule

Each format adapter must document supported variants, type mappings, metadata preservation, encoding behavior, dependencies, and limitations. Do not label an adapter “supported” until it passes fixture-based tests.

## Planned formats

| Format | Proposed library/approach | Important notes |
|---|---|---|
| Legacy Excel `.xls` | libxls | Reads old binary OLE/BIFF Excel files. Import directly to the canonical DataFrame model. Optional CSV conversion command may be offered separately. |
| Ordinary `.xlsx` | ZIP/OPC package reader + RapidXML for XML parts | RapidXML alone is not a workbook reader. Need ZIP extraction, workbook/worksheet relationship resolution, shared strings, inline strings, numeric/date handling, booleans, formulas/cached values, sparse cells, and styles where needed. |
| Password-encrypted `.xlsx` | Dedicated Office encryption/decryption layer; tiny-AES-c may be evaluated as a primitive only | Office password encryption is not “AES decrypt the file.” Must parse OLE Compound File streams, EncryptionInfo, derive keys, verify passwords, decrypt the package, and handle encryption variants. See security section. |
| XLSX writer | libxlsxwriter | C library for creating `.xlsx`; requires zlib. Writing is not the same as preserving every feature of a source workbook. |
| CSV | libcsv | Robust CSV parsing. Define delimiter, quoting, encoding, newline, header, type inference, missing tokens, and malformed-row policy. |
| TSV | Shared delimited-text layer, tab delimiter | Do not assume TSV has no quoting; provide explicit documented behavior. |
| FWF | Raku parser over byte/character slices, or a dedicated tested parser | Fixed widths can mean byte widths or character widths; make the mode explicit. |
| SAS | ReadStat | Plan for SAS7BDAT and XPORT reading/writing as supported by the library; metadata/catalog support is distinct from dataset support. |
| Stata | ReadStat | Version coverage and writer support must be checked against the selected ReadStat release. |
| SPSS | ReadStat | SAV, ZSAV, and POR variants; document supported read/write subsets. |
| R data | librdata | RData/RDS support is limited to supported objects/types; not a promise to reproduce arbitrary R object graphs. |

## libxls

Upstream: https://github.com/libxls/libxls

Use a maintained libxls 1.5.x release or later stable release after reviewing its current security and compatibility notes. 

Prefer direct cell extraction into the canonical model so that numbers, strings, blanks, dates, and workbook metadata can be handled deliberately. A separate `.xls` to CSV conversion command can be useful, but should not be the only import path.

## XLSX package/XML reader

Upstream RapidXML: https://rapidxml.sourceforge.net/

A normal `.xlsx` is a ZIP-based package. The reader must:
1. Detect and validate the package.
2. Open ZIP members using a maintained ZIP implementation.
3. Parse `[Content_Types].xml`, package relationships, workbook relationships, worksheets, shared strings, and styles as needed.
4. Resolve worksheet names and relationship targets rather than assuming fixed filenames.
5. Handle sparse cell coordinates and absent cells.
6. Interpret cell types and cached formula values with documented behavior.
7. Translate Excel date serials using workbook date-system settings.
8. Enforce limits on expanded ZIP size, entry count, XML size, and nesting.
9. Close all resources on errors.

RapidXML is an XML parser, not a ZIP library or spreadsheet semantic engine. Wrap it behind a C ABI if required for NativeCall interoperability.

## Password-encrypted XLSX: feasibility warning

Upstream tiny-AES-c: https://github.com/kokke/tiny-AES-c

tiny-AES-c implements AES block-cipher modes, but Office encryption requires more than the cipher primitive. A realistic design needs:
- OLE Compound File Binary (CFB) container parsing.
- Extraction of `EncryptionInfo` and `EncryptedPackage`.
- Support for specific encryption schemes and their version metadata.
- Password-to-key derivation, including the required hash algorithm and iteration scheme.
- Password/verifier validation and correct decryption of the package.
- ZIP/package processing after decryption.
- Secure handling of passwords and decrypted temporary data.

Office documents can use Standard, Agile, and other encryption approaches; algorithms and structures differ. See Microsoft's official specification: https://learn.microsoft.com/en-us/openspecs/office_file_formats/ms-offcrypto/

**Recommendation:** do not make tiny-AES-c the sole crypto plan. Before implementation, decide whether to use a well-reviewed Office-compatible decryption implementation or implement a narrowly scoped, specification-tested subset. Consider OpenSSL or another maintained crypto library for primitives where appropriate. If tiny-AES-c remains a requirement, it should be a replaceable primitive adapter and must pass independent test vectors. Explicitly document which encrypted workbook variants are supported; reject unsupported schemes safely.

Do not confuse workbook-open password encryption with worksheet/workbook structure protection.

## CSV and TSV

libcsv upstream: https://github.com/rgamble/libcsv

Normalize parsed fields through a schema/type inference layer. Provide options for:
- Header row.
- Delimiter, quote and escape behavior where supported.
- Encoding and invalid byte policy.
- Empty string versus missing tokens.
- Type inference on/off and sampling limits.
- Explicit column types.
- Ragged rows and malformed quoting.
- Locale-sensitive decimal marks only when explicitly requested.

Do not treat the strings `NA`, `NULL`, `NaN`, or an empty field as missing universally without options; these can be real data.

## ReadStat

Upstream: https://github.com/WizardMac/ReadStat

ReadStat provides reading/writing for major SAS, Stata, and SPSS dataset formats. Dataset formats, metadata formats, and version support differ. Maintain a matrix of each extension, read/write status, tested version range, labels, missing-value metadata, dates, and value labels.

Preserve variable labels, value labels, formats, and user-missing definitions in column metadata where practical. Avoid pretending those metadata can be faithfully represented by plain strings alone.

## librdata

Upstream: https://github.com/WizardMac/librdata

librdata reads RData/RDS and can write supported R data structures. The target is tabular data that the library can represent—not arbitrary R language objects, environments, functions, or all attributes. Verify its current API, maintenance state, build dependencies, and behavior before committing to write support.

## libxlsxwriter

Upstream: https://github.com/jmcnamara/libxlsxwriter

Use for `.xlsx` creation. Define mapping from DataFrame types to Excel cells, missing values, dates, strings, booleans, and numeric values. Expose a small first-release subset of formatting/options. Formula writing and cached results, worksheet naming, row/column limits, and error codes need tests.

