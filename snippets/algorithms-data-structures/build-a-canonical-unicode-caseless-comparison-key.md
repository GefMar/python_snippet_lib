---
title: "Build a Canonical Unicode Caseless Comparison Key"
snippet_type: algorithm
use_cases:
  - interoperability
  - validation
tested_python:
  - "3.14"
dependencies: []
related: []
---

# Build a Canonical Unicode Caseless Comparison Key

## Idea and Problem

Create a normalized casefold key that makes canonically equivalent Unicode text compare equal without changing the displayed value.

Lowercasing alone is not sufficient for general Unicode caseless equality, and
case folding does not always preserve a normalization form. Canonical caseless
matching decomposes the input, applies `casefold`, and decomposes the result
again. Store or display the original text and use only the derived key for
comparison or indexing.

## When to Use

Use this key when text from different systems should compare without case or
canonical-composition differences. It fits deduplication and equality lookup
when locale-independent Unicode semantics are acceptable. Choose a collation
library for language-aware ordering, and define a separate security policy for
usernames, identifiers, or visually confusable characters.

## Implementation

```python
from unicodedata import normalize


def canonical_caseless_key(text: str) -> str:
    decomposed = normalize("NFD", text)
    return normalize("NFD", decomposed.casefold())
```

## Example

```python
equivalent_pairs = (
    ("\u00e9", "e\u0301"),
    ("Stra\u00dfe", "STRASSE"),
    ("\u1fc3", "\u0397\u0399"),
)

matches = tuple(
    canonical_caseless_key(left) == canonical_caseless_key(right)
    for left, right in equivalent_pairs
)
compatibility_form_remains_distinct = (
    canonical_caseless_key("\uff21") != canonical_caseless_key("A")
)

assert (
    matches,
    compatibility_form_remains_distinct,
    canonical_caseless_key(""),
) == ((True, True, True), True, "")
```

## Trade-offs and Limitations

The returned key is decomposed text intended for comparison, not presentation.
Canonical caseless matching does not perform locale-aware collation, remove
accents, apply general compatibility normalization, or detect homoglyphs.
Case folding itself maps some compatibility characters, but not every
compatibility-equivalent form. Distinct inputs can intentionally share a key,
so a keyed mapping needs an explicit collision policy. Do not treat this
function as sufficient validation for security-sensitive identifiers.

## Related Snippets

<!-- catalog:related:start -->
No related snippets.
<!-- catalog:related:end -->
