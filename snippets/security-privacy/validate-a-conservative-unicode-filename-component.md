---
title: "Validate a Conservative Unicode Filename Component"
snippet_type: recipe
use_cases:
  - validation
  - interoperability
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/build-a-canonical-unicode-caseless-comparison-key.md
  - ../storage-databases/replace-a-file-atomically-with-a-sibling-temporary-file.md
  - ../storage-databases/store-bytes-by-their-content-digest.md
---

# Validate a Conservative Unicode Filename Component

## Idea and Problem

Normalize and validate one user-facing Unicode filename component without treating it as an authorized storage path.

Normalization runs first so compatibility characters cannot bypass later
checks. The validator rejects invalid input instead of silently deleting or
replacing characters: both path separators, non-space whitespace, Unicode
control-like categories, Windows-forbidden punctuation, trailing dots or
spaces, dot-only values, reserved device stems, and names above an explicit
UTF-8 byte budget all fail visibly.

## When to Use

Use this boundary when an application wants to retain a readable display name
from an untrusted upload or export request. The caller must supply a positive
byte budget appropriate to its own metadata and interoperability contract.
Store the file under a generated identifier inside a separately trusted root,
and keep the validated component as metadata; do not join an untrusted display
name directly into a storage path.

## Implementation

```python
import unicodedata


_WINDOWS_FORBIDDEN = frozenset('<>:"|?*')
_WINDOWS_RESERVED_STEMS = frozenset(
    {
        "AUX",
        "CLOCK$",
        "CON",
        "CONIN$",
        "CONOUT$",
        "NUL",
        "PRN",
    }
    | {f"COM{digit}" for digit in "123456789"}
    | {f"LPT{digit}" for digit in "123456789"}
)


def validate_unicode_filename_component(
    filename: str,
    *,
    max_utf8_bytes: int = 255,
) -> str:
    if not isinstance(filename, str):
        raise TypeError("filename must be text")
    if isinstance(max_utf8_bytes, bool) or not isinstance(max_utf8_bytes, int):
        raise TypeError("max_utf8_bytes must be an integer")
    if max_utf8_bytes <= 0:
        raise ValueError("max_utf8_bytes must be positive")

    normalized = unicodedata.normalize("NFKC", filename)
    if not normalized or all(character == "." for character in normalized):
        raise ValueError("filename must contain more than dots")
    if "/" in normalized or "\\" in normalized:
        raise ValueError("filename must be one path component")

    for character in normalized:
        if character.isspace() and character != " ":
            raise ValueError("filename contains non-space whitespace")
        if unicodedata.category(character).startswith("C"):
            raise ValueError("filename contains a disallowed Unicode category")
        if character in _WINDOWS_FORBIDDEN:
            raise ValueError("filename contains Windows-forbidden punctuation")

    if normalized.endswith((".", " ")):
        raise ValueError("filename must not end with a dot or space")

    stem = normalized.partition(".")[0].rstrip(" ").casefold().upper()
    if stem in _WINDOWS_RESERVED_STEMS:
        raise ValueError("filename uses a reserved device stem")

    if len(normalized.encode("utf-8")) > max_utf8_bytes:
        raise ValueError("filename exceeds the UTF-8 byte limit")
    return normalized
```

## Example

```python
normalized = validate_unicode_filename_component(
    "\uff32\u00e9sum\u00e9 \uff12\uff10\uff12\uff16.txt",
    max_utf8_bytes=64,
)


def is_rejected(candidate: str, *, max_utf8_bytes: int = 64) -> bool:
    try:
        validate_unicode_filename_component(
            candidate,
            max_utf8_bytes=max_utf8_bytes,
        )
    except ValueError:
        return True
    return False


assert (
    normalized,
    tuple(
        is_rejected(candidate, max_utf8_bytes=limit)
        for candidate, limit in (
            ("reports\uff0fsummary.txt", 64),
            ("report\tfinal.txt", 64),
            ("report\u200d.txt", 64),
            ("bad:name.txt", 64),
            ("report.txt.", 64),
            ("...", 64),
            ("COM\u00b9.log", 64),
            ("ninechars", 8),
        )
    ),
) == (
    "R\u00e9sum\u00e9 2026.txt",
    (True, True, True, True, True, True, True, True),
)
```

## Trade-offs and Limitations

NFKC intentionally merges some compatibility distinctions, and normalization
or case-insensitive filesystems can still create collisions. This policy does
not detect homoglyphs, guarantee uniqueness or containment, inspect file
content, or prevent filesystem races. Use an opaque generated storage ID, an
explicit collision policy, a trusted root, and race-safe creation or atomic
replacement separately. The reserved-name check is deliberately applied on
every operating system, which rejects some otherwise valid names. Other
filesystems can impose additional character, normalization, or length rules;
the UTF-8 budget is an application limit, not a universal portability claim.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Canonical Unicode Caseless Comparison Key](../algorithms-data-structures/build-a-canonical-unicode-caseless-comparison-key.md)
- [Replace a File Atomically with a Sibling Temporary File](../storage-databases/replace-a-file-atomically-with-a-sibling-temporary-file.md)
- [Store Bytes by Their Content Digest](../storage-databases/store-bytes-by-their-content-digest.md)
<!-- catalog:related:end -->
