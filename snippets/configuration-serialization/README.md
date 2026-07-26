# Configuration and Serialization

Configuration acquisition/layering or representation, schema, codec and
serialization semantics determine correctness.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Convert Decimal Values to Exact Minor Units](convert-decimal-values-to-exact-minor-units.md) | recipe | interoperability, validation | Convert a finite decimal value to an integer at an explicitly supplied scale without rounding or depending on ambient precision. |
| [Get Nested Values with a Validated Dot Path](get-nested-values-with-a-validated-dot-path.md) | recipe | configuration, parsing | Resolve a small dot-path grammar against JSON-like mappings and lists while keeping missing values distinct from malformed paths. |
| [Merge Nested Mappings Without Mutating Inputs](merge-nested-mappings-without-mutating-inputs.md) | recipe | configuration, data-transformation | Recursively merge colliding mappings into new dictionary containers while letting every non-mapping override replace the corresponding base value. |
| [Parse Compact Durations into timedelta](parse-compact-durations-into-timedelta.md) | recipe | configuration, parsing, validation | Parse one unsigned ASCII integer and one explicit lowercase unit into a timedelta under a deliberately small grammar. |
| [Parse Explicit Decimal and Binary Byte Sizes](parse-explicit-decimal-and-binary-byte-sizes.md) | recipe | configuration, parsing, validation | Parse case-sensitive decimal and binary unit suffixes into an exact non-negative integer number of bytes. |
| [Prune Empty Values from JSON-Like Data](prune-empty-values-from-json-like-data.md) | recipe | data-transformation, serialization | Remove explicitly defined empty values from generated JSON-like data while preserving meaningful falsy scalars and leaving the input untouched. |
| [Resolve an Absolute or Percentage Limit](resolve-an-absolute-or-percentage-limit.md) | recipe | configuration, parsing, validation | Resolve one strict configuration value as either an absolute count or a percentage of a known non-negative total. |
<!-- catalog:category:end -->
