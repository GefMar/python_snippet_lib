---
title: "Compute Inclusive and Exclusive Durations for a Bounded Nested Span Forest"
snippet_type: algorithm
use_cases:
  - data-transformation
  - observability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-a-validated-delta-between-cumulative-histogram-snapshots.md
  - measure-and-freeze-elapsed-time-in-a-context.md
  - ../data-processing/measure-time-in-a-state-within-a-half-open-window.md
---

# Compute Inclusive and Exclusive Durations for a Bounded Nested Span Forest

## Idea and Problem

Derive inclusive and self-only durations from a validated forest of properly nested half-open spans.

Inclusive duration is simply a span's end tick minus its start tick. Exclusive
duration removes the inclusive durations of direct children. Subtracting only
direct children is important: their inclusive durations already contain all
descendants, so subtracting grandchildren again would double-count them.

The calculation is valid only after proving that parents exist, ancestry is
acyclic, children are contained by their parent, and direct siblings do not
overlap. Those checks prevent a plausible-looking negative or inflated result
from malformed trace structure.

## When to Use

Use this algorithm for a bounded, already collected synchronous span forest
whose timestamps share one exact integer tick domain. It is useful when a
report needs to distinguish total time inside a span from time not attributed
to any nested child span.

Use a trace engine with explicit concurrency semantics when sibling operations
may overlap. Independent roots may overlap here because they represent
separate trees, but children of one parent must form non-overlapping half-open
intervals.

## Implementation

```python
import re
from dataclasses import dataclass
from itertools import pairwise

_MAX_SPANS = 4_096
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_SPAN_ID = re.compile(r"[A-Za-z][A-Za-z0-9_.:-]{0,63}", re.ASCII).fullmatch


@dataclass(frozen=True, slots=True)
class Span:
    span_id: str
    parent_id: str | None
    start: int
    end: int


@dataclass(frozen=True, slots=True)
class SpanDuration:
    span_id: str
    inclusive: int
    exclusive: int


def _validate_span_id(name: str, value: object) -> str:
    if type(value) is not str:
        raise TypeError(f"{name} must be an exact string")
    if _SPAN_ID(value) is None:
        raise ValueError(f"{name} is outside the supported grammar")
    return value


def _validate_tick(name: str, value: object) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not _MIN_INT64 <= value <= _MAX_INT64:
        raise ValueError(f"{name} is outside the signed-64 range")
    return value


def compute_span_durations(
    spans: tuple[Span, ...],
) -> tuple[SpanDuration, ...]:
    if type(spans) is not tuple:
        raise TypeError("spans must be an exact tuple")
    if not 1 <= len(spans) <= _MAX_SPANS:
        raise ValueError("span count is outside the supported range")

    by_id: dict[str, Span] = {}
    for span in spans:
        if type(span) is not Span:
            raise TypeError("spans must contain exact Span records")
        span_id = _validate_span_id("span_id", span.span_id)
        if span.parent_id is not None:
            _validate_span_id("parent_id", span.parent_id)
        start = _validate_tick("start", span.start)
        end = _validate_tick("end", span.end)
        if start >= end:
            raise ValueError("every span must have positive duration")
        if span_id in by_id:
            raise ValueError("span identifiers must be unique")
        by_id[span_id] = span

    children: dict[str, list[Span]] = {span_id: [] for span_id in by_id}
    for span in spans:
        parent_id = span.parent_id
        if parent_id is None:
            continue
        if parent_id == span.span_id:
            raise ValueError("a span cannot be its own parent")
        if parent_id not in by_id:
            raise ValueError("every named parent must exist")
        children[parent_id].append(span)

    state = {span_id: 0 for span_id in by_id}
    for first_id in by_id:
        if state[first_id] != 0:
            continue
        chain: list[str] = []
        current_id: str | None = first_id
        while current_id is not None and state[current_id] == 0:
            state[current_id] = 1
            chain.append(current_id)
            current_id = by_id[current_id].parent_id
        if current_id is not None and state[current_id] == 1:
            raise ValueError("span parent relationships contain a cycle")
        for span_id in chain:
            state[span_id] = 2

    child_durations: dict[str, int] = {}
    for parent_id, direct_children in children.items():
        parent = by_id[parent_id]
        for child in direct_children:
            if not (
                parent.start <= child.start
                and child.end <= parent.end
            ):
                raise ValueError("a child is not contained by its parent")

        ordered = sorted(
            direct_children,
            key=lambda child: (child.start, child.end, child.span_id),
        )
        for previous, current in pairwise(ordered):
            if previous.end > current.start:
                raise ValueError("direct sibling spans must not overlap")
        child_durations[parent_id] = sum(
            child.end - child.start for child in direct_children
        )

    return tuple(
        SpanDuration(
            span.span_id,
            span.end - span.start,
            span.end - span.start - child_durations[span.span_id],
        )
        for span in spans
    )
```

## Example

```python
spans = (
    Span("request", None, 0, 20),
    Span("fetch", "request", 2, 8),
    Span("decode", "fetch", 3, 5),
    Span("store", "request", 10, 14),
    Span("independent", None, 5, 9),
)

assert compute_span_durations(spans) == (
    SpanDuration("request", inclusive=20, exclusive=10),
    SpanDuration("fetch", inclusive=6, exclusive=4),
    SpanDuration("decode", inclusive=2, exclusive=2),
    SpanDuration("store", inclusive=4, exclusive=4),
    SpanDuration("independent", inclusive=4, exclusive=4),
)
```

## Trade-offs and Limitations

For `n` spans, validation and calculation use `O(n log n)` work in the worst
case because each sibling group is sorted, and `O(n)` auxiliary state. Output
preserves input order even though validation uses indexed parent lookups.

This model assumes complete, properly nested synchronous intervals. It does
not infer missing parents, repair clock skew, merge partial traces, calculate a
critical path, apportion overlapping asynchronous work, or capture timestamps.
The tick unit is deliberately opaque, and integer subtraction does not prove
that timestamps came from a monotonic clock.

Exclusive time includes every portion of a parent not covered by a direct
child, including gaps before, between, and after children. Overlapping roots
are permitted because no shared ownership relationship is implied between
separate trees.

## Related Snippets

<!-- catalog:related:start -->
- [Compute a Validated Delta Between Cumulative Histogram Snapshots](compute-a-validated-delta-between-cumulative-histogram-snapshots.md)
- [Measure and Freeze Elapsed Time in a Context](measure-and-freeze-elapsed-time-in-a-context.md)
- [Measure Time in a State Within a Half-Open Window](../data-processing/measure-time-in-a-state-within-a-half-open-window.md)
<!-- catalog:related:end -->
