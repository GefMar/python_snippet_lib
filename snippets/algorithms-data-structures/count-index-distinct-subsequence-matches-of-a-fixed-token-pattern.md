---
title: "Count Index-Distinct Subsequence Matches of a Fixed Token Pattern"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-a-longest-common-integer-subsequence-with-earliest-index-pair-ties.md
  - find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md
---

# Count Index-Distinct Subsequence Matches of a Fixed Token Pattern

## Idea and Problem

Count how many increasing selections of positions in a token stream spell one fixed pattern.

Keep one count for each prefix of the pattern. For an incoming token, add
the count for the previous prefix whenever that token can extend it. Update
the prefix counts backwards so an extension uses only counts from earlier
stream positions. This prevents one token from filling several positions in
the same match.

## When to Use

Use this to count all ordered occurrences of a short event pattern in a
finite stream. Both the stream and pattern use exact string tokens. Other
events can appear between matched tokens. Two matches are distinct when
they use different positions, even when their token values are identical.

## Implementation

```python
from collections.abc import Iterable


def count_subsequence_matches(tokens: Iterable[str], pattern: Iterable[str]) -> int:
    target = tuple(pattern)
    if any(type(token) is not str for token in target):
        raise TypeError("pattern tokens must be strings")
    if not target:
        return 1

    counts = [0] * (len(target) + 1)
    counts[0] = 1
    for token in tokens:
        if type(token) is not str:
            raise TypeError("stream tokens must be strings")
        for index in range(len(target) - 1, -1, -1):
            if token == target[index]:
                counts[index + 1] += counts[index]
    return counts[-1]
```

## Example

```python
events = ("open", "save", "open", "save")

assert (
    count_subsequence_matches(iter(events), ("open", "save")),
    count_subsequence_matches(events, ("open", "open")),
    count_subsequence_matches(events, ("save", "open", "open")),
    count_subsequence_matches((), ()),
) == (3, 1, 0, 1)
```

## Trade-offs and Limitations

Each stream token scans the pattern once. The counts use storage proportional
to the pattern length, and their integer sizes grow with the number of
matches. The empty pattern has one match, the empty selection, and returns
without consuming the stream.

Matches may overlap. For a contiguous match, use a substring search. For
session-specific matches, split the input into sessions before counting so
a match stays inside one session.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Longest Common Integer Subsequence with Earliest Index-Pair Ties](find-a-longest-common-integer-subsequence-with-earliest-index-pair-ties.md)
- [Find All Overlapping Exact Text Matches with Knuth-Morris-Pratt](find-all-overlapping-exact-text-matches-with-knuth-morris-pratt.md)
<!-- catalog:related:end -->
