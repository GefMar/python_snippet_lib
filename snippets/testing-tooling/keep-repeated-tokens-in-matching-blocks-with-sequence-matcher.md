---
title: "Keep Repeated Tokens in Matching Blocks with SequenceMatcher"
snippet_type: testing-technique
use_cases:
  - data-transformation
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - compare-a-bounded-text-capture-against-a-golden-fixture.md
  - ../algorithms-data-structures/build-a-canonical-minimum-unit-cost-edit-script-between-bounded-unicode-texts.md
---

# Keep Repeated Tokens in Matching Blocks with SequenceMatcher

## Idea and Problem

Find matching spans in two token sequences when frequently repeated tokens carry meaning in the comparison.

The default `SequenceMatcher` heuristic can exclude popular tokens as
starting points for matches. This can hide a shared run when it contains
only those tokens and has no adjacent matching anchor. Set `autojunk=False`
when repetitions should participate in the alignment.

## When to Use

Use this for a test or diagnostic that compares two small, fixed sequences of
hashable tokens, such as two versions of an event trace. The resulting
triples give the start index in each sequence and their shared span length.
Inspect those slices to explain which portions match.

## Implementation

```python
from collections.abc import Sequence
from difflib import SequenceMatcher


def matching_token_spans(
    before: Sequence[str], after: Sequence[str]
) -> tuple[tuple[int, int, int], ...]:
    matcher = SequenceMatcher(None, before, after, autojunk=False)
    return tuple(
        (block.a, block.b, block.size) for block in matcher.get_matching_blocks() if block.size
    )
```

## Example

```python
updates = ("progress",) * 200
before = ("start", *updates, "old summary")
after = ("open", "prepare", *updates, "new summary")

default_spans = tuple(
    block for block in SequenceMatcher(None, before, after).get_matching_blocks() if block.size
)
spans = matching_token_spans(before, after)
assert spans == ((1, 2, 200),)
start_before, start_after, size = spans[0]
assert (
    default_spans,
    before[start_before : start_before + size],
    after[start_after : start_after + size],
) == (
    (),
    updates,
    updates,
)
```

## Trade-offs and Limitations

The popularity heuristic is based on the second sequence and applies to
sequences of at least 200 items. Disabling it can make repetitive inputs
slower. `SequenceMatcher` has quadratic worst-case time, so use this with
small diagnostic sequences. The original ordering and exact token equality
determine which blocks can match.

The nonempty blocks describe one alignment. They do not guarantee a minimum
edit script or a unique correspondence between repeats. The helper discards
the zero-length sentinel that `get_matching_blocks()` appends.

## Related Snippets

<!-- catalog:related:start -->
- [Compare a Bounded Text Capture Against a Golden Fixture](compare-a-bounded-text-capture-against-a-golden-fixture.md)
- [Build a Canonical Minimum Unit-Cost Edit Script Between Bounded Unicode Texts](../algorithms-data-structures/build-a-canonical-minimum-unit-cost-edit-script-between-bounded-unicode-texts.md)
<!-- catalog:related:end -->
