---
title: "Map String Offsets to Line and Column Across Mixed Newlines"
snippet_type: recipe
use_cases:
  - data-transformation
  - interoperability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - map-utf-16-code-unit-offsets-to-python-string-boundaries.md
  - ../testing-tooling/replace-selected-python-name-tokens-without-touching-strings-or-comments.md
---

# Map String Offsets to Line and Column Across Mixed Newlines

## Idea and Problem

Translate between absolute Python string offsets and line-relative coordinates while preserving mixed newline spellings.

Index the start and content end of each line. Match CRLF before lone CR
or LF so it counts as one separator. A position just before a separator is
the end of its line, and a position just after it is the start of the next
line. The position between CR and LF has no line-column coordinate.

## When to Use

Use this when a parser returns offsets into one decoded string and diagnostic
or annotation code needs zero-based lines and columns. Preserve the original
newline spellings when you read the text. Build the index for the same string
revision that the offsets describe.

## Implementation

```python
from bisect import bisect_right
from dataclasses import dataclass
from re import finditer


@dataclass(frozen=True, slots=True)
class LineIndex:
    starts: tuple[int, ...]
    content_ends: tuple[int, ...]

    @classmethod
    def from_text(cls, text: str) -> LineIndex:
        starts = [0]
        ends = []
        for ending in finditer(r"\r\n|[\r\n]", text):
            ends.append(ending.start())
            starts.append(ending.end())
        ends.append(len(text))
        return cls(tuple(starts), tuple(ends))

    def position(self, offset: int) -> tuple[int, int]:
        if type(offset) is not int:
            raise TypeError("offset must be an integer")
        if not 0 <= offset <= self.content_ends[-1]:
            raise ValueError("offset lies outside the text")
        line = bisect_right(self.starts, offset) - 1
        if offset > self.content_ends[line]:
            raise ValueError("offset splits a CRLF separator")
        return line, offset - self.starts[line]

    def offset(self, line: int, column: int) -> int:
        if type(line) is not int or type(column) is not int:
            raise TypeError("line and column must be integers")
        if not 0 <= line < len(self.starts):
            raise ValueError("line lies outside the text")
        start = self.starts[line]
        if not 0 <= column <= self.content_ends[line] - start:
            raise ValueError("column lies outside the line")
        return start + column
```

## Example

```python
index = LineIndex.from_text("A\r\n\U0001f680\rZ\n")
try:
    index.position(2)
except ValueError:
    split_rejected = True
else:
    split_rejected = False

assert (
    index.position(1),
    index.position(3),
    index.offset(1, 1),
    index.position(7),
    LineIndex.from_text("").position(0),
    split_rejected,
) == ((0, 1), (1, 0), 4, (3, 0), (0, 0), True)
```

## Trade-offs and Limitations

The index takes one pass over the text and stores two offsets per line. Line
lookup takes logarithmically many comparisons, and reverse lookup accesses
one line directly. A trailing separator contributes a final empty line, and
empty text has one empty line.

This policy recognizes only CR, LF and CRLF. Columns count Python code points
rather than bytes, UTF-16 units or visual widths. Use the target protocol's
separator and column rules when they differ.

## Related Snippets

<!-- catalog:related:start -->
- [Map UTF-16 Code-Unit Offsets to Python String Boundaries](map-utf-16-code-unit-offsets-to-python-string-boundaries.md)
- [Replace Selected Python NAME Tokens without Touching Strings or Comments](../testing-tooling/replace-selected-python-name-tokens-without-touching-strings-or-comments.md)
<!-- catalog:related:end -->
