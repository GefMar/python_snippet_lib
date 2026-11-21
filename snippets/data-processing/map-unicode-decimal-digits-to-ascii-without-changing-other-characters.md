---
title: "Map Unicode Decimal Digits to ASCII Without Changing Other Characters"
snippet_type: recipe
use_cases:
  - data-transformation
  - interoperability
tested_python:
  - "3.14"
dependencies: []
related:
  - report-unicode-nfc-collisions-before-importing-keys.md
  - ../algorithms-data-structures/sort-text-by-literal-text-runs-and-ascii-digit-runs.md
---

# Map Unicode Decimal Digits to ASCII Without Changing Other Characters

## Idea and Problem

Convert decimal digits from different writing systems to ASCII without changing letters, punctuation, or other numeric symbols.

Ask the Unicode database for the decimal value of each character rather than testing whether it looks numeric. A superscript digit or a vulgar fraction is numeric in a broader sense but has no decimal digit value.

## When to Use

Use this when importing display text into a workflow that needs ASCII decimal digits and deliberately preserves everything else. It is useful before a separate, explicit step that interprets the text.

## Implementation

```python
from unicodedata import decimal


def ascii_decimal_digits(text: str) -> str:
    output: list[str] = []
    for char in text:
        value = decimal(char, None)
        output.append(char if value is None else str(value))
    return "".join(output)
```

## Example

```python
raw = "label \u0660\u0661\u0662; fullwidth \uff13\uff14; x²; ½ and 5"
converted = ascii_decimal_digits(raw)

assert (converted, ascii_decimal_digits(converted), ascii_decimal_digits("")) == (
    "label 012; fullwidth 34; x²; ½ and 5",
    "label 012; fullwidth 34; x²; ½ and 5",
    "",
)
```

## Trade-offs and Limitations

This is not a number parser, locale-aware normalization, or an identifier-safety check. It leaves decimal separators, signs, and nondecimal numeric characters unchanged. Mapping digits across scripts deliberately allows some distinct inputs to produce the same result.

## Related Snippets

<!-- catalog:related:start -->
- [Report Unicode NFC Collisions Before Importing Keys](report-unicode-nfc-collisions-before-importing-keys.md)
- [Sort Text by Literal Text Runs and ASCII Digit Runs](../algorithms-data-structures/sort-text-by-literal-text-runs-and-ascii-digit-runs.md)
<!-- catalog:related:end -->
