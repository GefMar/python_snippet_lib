---
title: "Rank Bounded Records with Stable Ties and Neighbor Windows"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - sort-dotted-release-labels-with-an-explicit-last-marker.md
  - ../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md
  - ../data-processing/yield-stream-items-with-bounded-neighbor-context.md
---

# Rank Bounded Records with Stable Ties and Neighbor Windows

## Idea and Problem

Order a bounded tuple by descending finite score, retain declaration order for ties, and attach a clipped rank-neighbor window to each top result.

Every output receives its one-based position in the complete ordering. A
subject's neighbors come from that same ordering, exclude the subject, and may
extend beyond the returned top records. Fixed input, radius, and neighbor-link
limits keep validation and output construction predictable.

## When to Use

Use this algorithm when scores have already been calculated, input order is
the intended tie-breaker, and consumers need both a top slice and nearby
positions. The complete input must fit in memory, identifiers must be unique,
and an ordinal one-based position must be the desired meaning of rank even
when scores are equal.

Choose a different rule when ties should share a rank, all tied records must be
included past `top_k`, or neighborhood depends on something other than sorted
position.

## Implementation

```python
import math
import re
from dataclasses import dataclass

_MAX_RECORDS = 512
_MAX_IDENTIFIER_BYTES = 64
_MAX_RADIUS = 64
_MAX_NEIGHBOR_LINKS = 4_096
_IDENTIFIER_PATTERN = re.compile(r"[A-Za-z0-9][A-Za-z0-9_.-]*")


@dataclass(frozen=True, slots=True)
class ScoreRecord:
    identifier: str
    score: int | float


@dataclass(frozen=True, slots=True)
class RankNeighbor:
    identifier: str
    score: int | float
    rank: int


@dataclass(frozen=True, slots=True)
class RankedRecord:
    identifier: str
    score: int | float
    rank: int
    neighbors: tuple[RankNeighbor, ...]


def _require_identifier(identifier: object) -> str:
    if type(identifier) is not str:
        raise TypeError("identifier must be an exact str")
    if (
        not identifier.isascii()
        or len(identifier.encode("ascii")) > _MAX_IDENTIFIER_BYTES
        or _IDENTIFIER_PATTERN.fullmatch(identifier) is None
    ):
        raise ValueError("identifier has an invalid format or exceeds the byte limit")
    return identifier


def _require_score(score: object) -> int | float:
    if type(score) not in (int, float):
        raise TypeError("score must be an exact int or float")
    if type(score) is float and not math.isfinite(score):
        raise ValueError("score must be finite")
    return score


def rank_with_neighbors(
    records: tuple[ScoreRecord, ...],
    *,
    top_k: int,
    radius: int,
) -> tuple[RankedRecord, ...]:
    if type(records) is not tuple:
        raise TypeError("records must be an exact tuple")
    if not 1 <= len(records) <= _MAX_RECORDS:
        raise ValueError("record count is outside the supported range")
    if type(top_k) is not int:
        raise TypeError("top_k must be an exact int")
    if not 1 <= top_k <= len(records):
        raise ValueError("top_k must be between 1 and the record count")
    if type(radius) is not int:
        raise TypeError("radius must be an exact int")
    if not 0 <= radius <= _MAX_RADIUS:
        raise ValueError("radius is outside the supported range")

    seen_identifiers: set[str] = set()
    validated: list[tuple[int, ScoreRecord]] = []
    for declaration_index, record in enumerate(records):
        if type(record) is not ScoreRecord:
            raise TypeError("every record must be an exact ScoreRecord")
        identifier = _require_identifier(record.identifier)
        if identifier in seen_identifiers:
            raise ValueError("record identifiers must be unique")
        seen_identifiers.add(identifier)
        _require_score(record.score)
        validated.append((declaration_index, record))

    neighbor_link_count = sum(
        min(len(records), index + radius + 1) - max(0, index - radius) - 1
        for index in range(top_k)
    )
    if neighbor_link_count > _MAX_NEIGHBOR_LINKS:
        raise ValueError("requested neighbor output exceeds the work limit")

    ordered = tuple(
        record
        for _declaration_index, record in sorted(
            validated,
            key=lambda item: (-item[1].score, item[0]),
        )
    )

    results: list[RankedRecord] = []
    for subject_index, subject in enumerate(ordered[:top_k]):
        start = max(0, subject_index - radius)
        stop = min(len(ordered), subject_index + radius + 1)
        neighbors = tuple(
            RankNeighbor(
                identifier=ordered[index].identifier,
                score=ordered[index].score,
                rank=index + 1,
            )
            for index in range(start, stop)
            if index != subject_index
        )
        results.append(
            RankedRecord(
                identifier=subject.identifier,
                score=subject.score,
                rank=subject_index + 1,
                neighbors=neighbors,
            )
        )

    return tuple(results)
```

## Example

```python
records = (
    ScoreRecord("alpha", 7.0),
    ScoreRecord("beta", 10.0),
    ScoreRecord("gamma", 10.0),
    ScoreRecord("delta", 8),
    ScoreRecord("epsilon", 7.0),
)
unchanged = tuple(records)

ranked = rank_with_neighbors(records, top_k=3, radius=1)

try:
    rank_with_neighbors(
        (ScoreRecord("same", 2.0), ScoreRecord("same", 1.0)),
        top_k=1,
        radius=0,
    )
except ValueError:
    duplicate_rejected = True
else:
    duplicate_rejected = False

try:
    rank_with_neighbors(
        (ScoreRecord("invalid", float("nan")),),
        top_k=1,
        radius=0,
    )
except ValueError:
    non_finite_rejected = True
else:
    non_finite_rejected = False

assert (
    ranked,
    records == unchanged,
    duplicate_rejected,
    non_finite_rejected,
) == (
    (
        RankedRecord(
            "beta",
            10.0,
            1,
            (RankNeighbor("gamma", 10.0, 2),),
        ),
        RankedRecord(
            "gamma",
            10.0,
            2,
            (
                RankNeighbor("beta", 10.0, 1),
                RankNeighbor("delta", 8, 3),
            ),
        ),
        RankedRecord(
            "delta",
            8,
            3,
            (
                RankNeighbor("gamma", 10.0, 2),
                RankNeighbor("alpha", 7.0, 4),
            ),
        ),
    ),
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Sorting takes `O(n log n)` time and `O(n)` auxiliary memory. Constructing the
result takes one immutable object per accepted neighbor link, capped at 4,096;
neighbors repeat across overlapping windows. The function validates every
record before sorting and never mutates the input tuple or its frozen records.

Integer scores may be arbitrarily large, while float scores must be finite.
Numerically equal integers and floats tie, signed zeroes tie, and declaration
order resolves every tie. `top_k` is a strict slice rather than a tie-expanded
cutoff. The algorithm does not calculate scores, aggregate observations over
time, read from a database, define a fairness policy, or make a
domain-specific recommendation.

## Related Snippets

<!-- catalog:related:start -->
- [Sort Dotted Release Labels with an Explicit Last Marker](sort-dotted-release-labels-with-an-explicit-last-marker.md)
- [Select One Record per Key with an Explicit Ranking Rule](../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md)
- [Yield Stream Items with Bounded Neighbor Context](../data-processing/yield-stream-items-with-bounded-neighbor-context.md)
<!-- catalog:related:end -->
