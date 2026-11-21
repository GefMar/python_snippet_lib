---
title: "Wrap Prose Lines Without Collapsing Explicit Line Breaks"
snippet_type: recipe
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/wrap-bounded-ascii-words-with-minimum-squared-raggedness.md
  - ../python-language/normalize-raw-docstring-indentation-with-inspect-cleandoc.md
---

# Wrap Prose Lines Without Collapsing Explicit Line Breaks

## Idea and Problem

Add wrapping inside existing prose lines without merging separate paragraphs or breaking long tokens.

Apply a reusable text wrapper separately to each line in the source. Split by explicit line feeds so a final line feed and blank lines still act as separators.

## When to Use

Use this for plain LF-delimited text where hard breaks have meaning. Long words and hyphenated identifiers are kept intact even when one token is wider than the requested width.

## Implementation

```python
from textwrap import TextWrapper


def wrap_lines(text: str, width: int) -> str:
    if type(width) is not int or width < 1:
        raise ValueError("width must be a positive integer")
    wrapper = TextWrapper(width=width, break_long_words=False, break_on_hyphens=False)
    return "\n".join(wrapper.fill(line) for line in text.split("\n"))
```

## Example

```python
source = "alpha beta gamma\n\nlong-identifier value\n"
assert (wrap_lines(source, 10), wrap_lines("", 10)) == (
    "alpha beta\ngamma\n\nlong-identifier\nvalue\n",
    "",
)
```

## Trade-offs and Limitations

The width counts Python string positions, not terminal display cells. The wrapper normalizes some whitespace within each line. Do not use this for preformatted tables, byte limits, or when exact whitespace must survive.

## Related Snippets

<!-- catalog:related:start -->
- [Wrap Bounded ASCII Words with Minimum Squared Raggedness](../algorithms-data-structures/wrap-bounded-ascii-words-with-minimum-squared-raggedness.md)
- [Normalize Raw Docstring Indentation with inspect.cleandoc](../python-language/normalize-raw-docstring-indentation-with-inspect-cleandoc.md)
<!-- catalog:related:end -->
