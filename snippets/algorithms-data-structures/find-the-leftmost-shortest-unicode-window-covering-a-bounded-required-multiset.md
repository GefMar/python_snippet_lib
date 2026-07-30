---
title: "Find the Leftmost Shortest Unicode Window Covering a Bounded Required Multiset"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-every-anagram-window-in-bounded-unicode-text.md
  - find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md
  - find-the-leftmost-shortest-non-empty-contiguous-integer-span-reaching-a-sum-threshold.md
---

# Find the Leftmost Shortest Unicode Window Covering a Bounded Required Multiset

## Idea and Problem

Find the shortest contiguous text window that contains every required Unicode code point with its requested multiplicity.

A sliding window expands until it covers the complete multiset and then shrinks
from the left while coverage remains valid. Tracking a single missing-
multiplicity count makes the coverage test constant expected time. Each source
position enters and leaves the window at most once.

Repeated requirements matter: a window covering `"aab"` needs two `"a"` code
points, not merely the set `{"a", "b"}`. Equal-length solutions are resolved by
the smallest source start index.

## When to Use

Use this algorithm for bounded, fully materialized Unicode text when exact
code-point membership matters and order inside the returned window does not.
It fits compact token-inventory searches, minimum context extraction, and
deterministic validation examples with duplicate requirements.

Normalize or case-fold both inputs before calling only when that transformation
is explicitly part of the surrounding contract. Python string indices count
code points, not UTF-8 bytes or user-perceived grapheme clusters.

## Implementation

```python
from collections import Counter
from dataclasses import dataclass

_MAX_SOURCE_CODE_POINTS = 8_192
_MAX_SOURCE_UTF8_BYTES = 16_384
_MAX_REQUIRED_CODE_POINTS = 256
_MAX_REQUIRED_UTF8_BYTES = 512


@dataclass(frozen=True, slots=True)
class TextWindow:
    start: int
    stop: int


def _validate_text(
    name: str,
    value: object,
    *,
    max_code_points: int,
    max_utf8_bytes: int,
    allow_empty: bool,
) -> str:
    if type(value) is not str:
        raise TypeError(f"{name} must be an exact string")
    if not allow_empty and not value:
        raise ValueError(f"{name} must not be empty")
    if len(value) > max_code_points:
        raise ValueError(f"{name} exceeds its code-point limit")
    utf8_bytes = 0
    for character in value:
        code_point = ord(character)
        if 0xD800 <= code_point <= 0xDFFF:
            raise ValueError(f"{name} contains a surrogate code point")
        if code_point <= 0x7F:
            utf8_bytes += 1
        elif code_point <= 0x7FF:
            utf8_bytes += 2
        elif code_point <= 0xFFFF:
            utf8_bytes += 3
        else:
            utf8_bytes += 4
        if utf8_bytes > max_utf8_bytes:
            raise ValueError(f"{name} exceeds its UTF-8 byte limit")
    return value


def leftmost_shortest_multiset_window(
    source: str,
    required: str,
) -> TextWindow | None:
    source = _validate_text(
        "source",
        source,
        max_code_points=_MAX_SOURCE_CODE_POINTS,
        max_utf8_bytes=_MAX_SOURCE_UTF8_BYTES,
        allow_empty=True,
    )
    required = _validate_text(
        "required",
        required,
        max_code_points=_MAX_REQUIRED_CODE_POINTS,
        max_utf8_bytes=_MAX_REQUIRED_UTF8_BYTES,
        allow_empty=False,
    )
    if len(required) > len(source):
        return None

    needed = Counter(required)
    present: dict[str, int] = {}
    missing = len(required)
    left = 0
    best: TextWindow | None = None

    for right, character in enumerate(source):
        if character in needed:
            previous = present.get(character, 0)
            present[character] = previous + 1
            if previous < needed[character]:
                missing -= 1

        while missing == 0:
            candidate = TextWindow(left, right + 1)
            if best is None or (
                candidate.stop - candidate.start,
                candidate.start,
            ) < (best.stop - best.start, best.start):
                best = candidate

            outgoing = source[left]
            if outgoing in needed:
                present[outgoing] -= 1
                if present[outgoing] < needed[outgoing]:
                    missing += 1
            left += 1

    return best
```

## Example

```python
text = "cabéxxabéc"
window = leftmost_shortest_multiset_window(text, "éab")
assert window == TextWindow(1, 4)
assert text[window.start : window.stop] == "abé"

repeated = "xy🙂x🙂y"
repeat_window = leftmost_shortest_multiset_window(repeated, "🙂🙂y")
assert repeat_window == TextWindow(1, 5)
assert leftmost_shortest_multiset_window("e\N{COMBINING ACUTE ACCENT}", "é") is None
```

## Trade-offs and Limitations

For source length `n`, requirement length `m`, and `u` distinct required code
points, the algorithm uses `O(n + m)` time and `O(u)` auxiliary state. The
returned half-open indices slice the original Python string directly without
constructing each candidate substring.

Matching is exact and normalization-sensitive. A precomposed character and a
visually similar combining sequence are different code-point multisets.
Likewise, the algorithm does not case-fold, segment grapheme clusters, tokenize
words, apply locale rules, or perform approximate matching.

Both code-point and UTF-8 byte caps are enforced because storage and traversal
budgets can differ. The function materializes the complete inputs and is not a
streaming search. An empty requirement is rejected rather than defining every
boundary as an equally short zero-width solution.

## Related Snippets

<!-- catalog:related:start -->
- [Find Every Anagram Window in Bounded Unicode Text](find-every-anagram-window-in-bounded-unicode-text.md)
- [Find All Overlapping Exact Text Matches with Knuth-Morris-Pratt](find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md)
- [Find the Leftmost Shortest Non-Empty Contiguous Integer Span Reaching a Sum Threshold](find-the-leftmost-shortest-non-empty-contiguous-integer-span-reaching-a-sum-threshold.md)
<!-- catalog:related:end -->
