# Configuration and Serialization

Configuration acquisition/layering or representation, schema, codec and
serialization semantics determine correctness.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Convert Decimal Values to Exact Minor Units](convert-decimal-values-to-exact-minor-units.md) | recipe | interoperability, validation | Convert a finite decimal value to an integer at an explicitly supplied scale without rounding or depending on ambient precision. |
| [Get Nested Values with a Validated Dot Path](get-nested-values-with-a-validated-dot-path.md) | recipe | configuration, parsing | Resolve a small dot-path grammar against JSON-like mappings and lists while keeping missing values distinct from malformed paths. |
<!-- catalog:category:end -->
