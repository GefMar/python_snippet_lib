---
title: "Sort Text by Literal Text Runs and ASCII Digit Runs"
snippet_type: recipe
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/parse-and-compare-bounded-semantic-versions-2-0-0.md
  - rank-bounded-records-with-stable-ties-and-neighbor-windows.md
---

# Sort Text by Literal Text Runs and ASCII Digit Runs

## Idea and Problem

Sort strings such as batch filenames by the numeric values embedded in their literal text.

Split each string into maximal ASCII digit runs and runs of other characters.
For a number, compare the length of its significant digits, then those digits
in string order. This compares unsigned decimal values without converting them
to integers, including very long numbers. Compare text runs literally.

## When to Use

Use this for a stable display order of labels such as `run-2.csv` and
`run-10.csv`. Equal numerical parts may have different zero padding. When
all other parts also compare equal, Python's stable sort keeps the original
order. At a position with different run kinds, a numeric run comes first.

## Implementation

```python
import re


_RUNS = re.compile(r"[0-9]+|[^0-9]+")


def literal_run_key(text: str) -> tuple[tuple[int, int, str], ...]:
    """Return a sort key for ASCII numbers and literal text runs."""
    if type(text) is not str:
        raise TypeError("text must be a string")

    parts: list[tuple[int, int, str]] = []
    for match in _RUNS.finditer(text):
        run = match.group()
        if "0" <= run[0] <= "9":
            significant = run.lstrip("0") or "0"
            parts.append((0, len(significant), significant))
        else:
            parts.append((1, 0, run))
    return tuple(parts)
```

## Example

```python
names = ("run-10.csv", "run-002.csv", "run-2.csv", "run-000.csv", "Run-2.csv", "run-2a.csv")
many_nines = "run-" + "9" * 5_000
next_power = "run-1" + "0" * 5_000

assert (
    sorted(names, key=literal_run_key),
    sorted((next_power, many_nines), key=literal_run_key),
) == (
    ["Run-2.csv", "run-000.csv", "run-002.csv", "run-2.csv", "run-2a.csv", "run-10.csv"],
    [many_nines, next_power],
)
```

## Trade-offs and Limitations

The sort stores one key per string, with additional storage proportional to
the input text. The rule is case-sensitive and compares text by Python's
ordinary string ordering. Non-ASCII digits remain literal text characters.

Signs, separators and decimal points are text. For decimal numbers, locale
collation or version precedence, use a parser or comparator for that grammar.

## Related Snippets

<!-- catalog:related:start -->
- [Parse and Compare Bounded Semantic Versions 2.0.0](../configuration-serialization/parse-and-compare-bounded-semantic-versions-2-0-0.md)
- [Rank Bounded Records with Stable Ties and Neighbor Windows](rank-bounded-records-with-stable-ties-and-neighbor-windows.md)
<!-- catalog:related:end -->
