---
title: "Sort Dotted Release Labels with an Explicit Last Marker"
snippet_type: algorithm
use_cases:
  - parsing
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - resolve-stable-ordering-constraints-with-topological-sort.md
  - ../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md
  - ../configuration-serialization/parse-explicit-decimal-and-binary-byte-sizes.md
---

# Sort Dotted Release Labels with an Explicit Last Marker

## Idea and Problem

Sort bounded dotted numeric labels component by component while placing explicit whole-value last markers after every numeric label.

Lexical ordering puts `1.10` before `1.2`, even though the numeric components
imply the opposite order. A tagged key separates ordinary component tuples
from the `last` and `latest` aliases, so mixed value types never need direct
comparison and all marker spellings share one stable tie.

## When to Use

Use this algorithm for a small domain that deliberately accepts only canonical
ASCII labels such as `2`, `2.4`, or `10.1.3`, plus a terminal marker. The input
must be a finite iterable, and component-wise tuple ordering must be the exact
domain rule. Use a SemVer or PEP 440 implementation when prereleases, build
metadata, epochs, compatible releases, or package dependency decisions matter.

## Implementation

```python
from collections.abc import Iterable


ReleaseSortKey = tuple[int, tuple[int, ...]]

_LAST_MARKERS = frozenset({"last", "latest"})
_MAX_LABEL_CHARACTERS = 128
_MAX_COMPONENTS = 16
_MAX_COMPONENT_DIGITS = 32


def dotted_release_sort_key(label: str) -> ReleaseSortKey:
    if not isinstance(label, str):
        raise TypeError("release label must be text")
    if not label or len(label) > _MAX_LABEL_CHARACTERS or not label.isascii():
        raise ValueError("release label must be non-empty bounded ASCII text")

    if label.lower() in _LAST_MARKERS:
        return (1, ())

    components = label.split(".")
    if len(components) > _MAX_COMPONENTS:
        raise ValueError("release label has too many components")

    numeric_components: list[int] = []
    for component in components:
        if not component or not component.isdecimal():
            raise ValueError("release components must contain only ASCII decimal digits")
        if len(component) > _MAX_COMPONENT_DIGITS:
            raise ValueError("release component is too long")
        if len(component) > 1 and component.startswith("0"):
            raise ValueError("release components must not contain leading zeroes")
        numeric_components.append(int(component))

    return (0, tuple(numeric_components))


def sort_dotted_release_labels(labels: Iterable[str]) -> list[str]:
    if isinstance(labels, str):
        raise TypeError("labels must be an iterable of labels, not one label")
    return sorted(labels, key=dotted_release_sort_key)
```

## Example

```python
labels = [
    "2.0",
    "1.10",
    "LAST",
    "1.2",
    "latest",
    "1.2.0",
    "10",
    "Last",
]
ordered = sort_dotted_release_labels(labels)

invalid_labels = (
    "",
    "01.2",
    "1.02",
    "1.",
    ".1",
    "1..2",
    "1-rc1",
    "v1.2",
    "last.1",
    "１.2",
    "1" * 33,
    ".".join("1" for _ in range(17)),
)
rejected = []
for label in invalid_labels:
    try:
        dotted_release_sort_key(label)
    except ValueError:
        rejected.append(label)

try:
    dotted_release_sort_key(12)  # type: ignore[arg-type]
except TypeError:
    non_text_rejected = True
else:
    non_text_rejected = False

assert (
    ordered,
    tuple(rejected),
    non_text_rejected,
    dotted_release_sort_key("2.10"),
    dotted_release_sort_key("last") == dotted_release_sort_key("LATEST"),
) == (
    ["1.2", "1.2.0", "1.10", "2.0", "10", "LAST", "latest", "Last"],
    invalid_labels,
    True,
    (0, (2, 10)),
    True,
)
```

## Trade-offs and Limitations

Sorting uses `O(n)` additional memory and `O(n log n)` key comparisons. A
shorter component tuple sorts before an otherwise equal extension, so `1`
precedes `1.0`; trailing zero components are not normalized away. All accepted
spellings of `last` and `latest` compare equal and retain their input order.
The fixed character, component, and digit limits intentionally reject broader
version syntaxes. This is a local label convention, not SemVer, PEP 440, or a
safe basis for dependency compatibility decisions.

## Related Snippets

<!-- catalog:related:start -->
- [Resolve Stable Ordering Constraints with Topological Sort](resolve-stable-ordering-constraints-with-topological-sort.md)
- [Select One Record per Key with an Explicit Ranking Rule](../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md)
- [Parse Explicit Decimal and Binary Byte Sizes](../configuration-serialization/parse-explicit-decimal-and-binary-byte-sizes.md)
<!-- catalog:related:end -->
