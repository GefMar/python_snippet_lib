---
title: "Rank Hierarchy Paths with Bounded Weighted Edit Distance"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-canonical-unicode-caseless-comparison-key.md
  - match-strict-mutual-nearest-neighbors-with-a-comparison-budget.md
  - ../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md
---

# Rank Hierarchy Paths with Bounded Weighted Edit Distance

## Idea and Problem

Rank bounded hierarchy candidates by the weighted sum of exact per-level Levenshtein distances without silently choosing one candidate.

Paths are immutable tuples of Unicode strings. A query at the full candidate
depth uses only direct alignment. A query shorter by exactly one level is
evaluated twice: once with a missing leading level and once with a missing
trailing level. The missing slot is represented by `None`, not a text sentinel,
and is not charged; at least two real query levels must still be compared.

Every candidate is evaluated before the maximum-distance filter is applied.
Eligible scores are ordered by distance and then candidate input position, and
all candidates at the best distance are returned as an explicit tie tuple.

## When to Use

Use this algorithm for a small hierarchy whose labels are approximately known,
whose levels have documented relative importance, and whose producer may omit
exactly one boundary level. The level weights correspond to the full candidate
depth. Treat an empty `best_ties` result or a multi-candidate tie as a request
for review; the function deliberately does not remap any records.

This comparison is exact at the Unicode code-point level: it performs no
normalization, case folding, transliteration, or locale-aware comparison.
Normalize before calling only when that is an explicit domain rule. Use a
search index or a domain-specific model for large candidate sets, internal
missing levels, token reordering, synonyms, or semantic similarity.

## Implementation

```python
from dataclasses import dataclass
from enum import StrEnum


_MIN_COMPARED_LEVELS = 2
_MAX_LEVELS = 8
_MAX_LEVEL_CHARACTERS = 128
_MAX_CANDIDATES = 256
_MAX_LEVEL_WEIGHT = 1_000_000
_MAX_DISTANCE = 1_000_000_000
_HARD_MAX_DP_CELLS = 1_000_000


class PathAlignment(StrEnum):
    DIRECT = "direct"
    QUERY_MISSING_LEADING = "query-missing-leading"
    QUERY_MISSING_TRAILING = "query-missing-trailing"


@dataclass(frozen=True, slots=True)
class HierarchyPathScore:
    candidate_index: int
    candidate: tuple[str, ...]
    distance: int
    alignments: tuple[PathAlignment, ...]


@dataclass(frozen=True, slots=True)
class HierarchyPathRanking:
    ranked: tuple[HierarchyPathScore, ...]
    best_ties: tuple[HierarchyPathScore, ...]
    evaluated_candidates: int
    dp_cells: int


def _validate_path(
    path: object,
    *,
    name: str,
    expected_depth: int | None = None,
) -> tuple[str, ...]:
    if type(path) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if not _MIN_COMPARED_LEVELS <= len(path) <= _MAX_LEVELS:
        raise ValueError(f"{name} depth is outside the supported range")
    if expected_depth is not None and len(path) != expected_depth:
        raise ValueError(f"{name} must have the full candidate depth")
    for level in path:
        if type(level) is not str:
            raise TypeError(f"every {name} level must be an exact string")
        if not level or len(level) > _MAX_LEVEL_CHARACTERS:
            raise ValueError(f"a {name} level is outside the supported range")
    return path


def _levenshtein(left: str, right: str) -> int:
    if len(left) < len(right):
        left, right = right, left
    previous = list(range(len(right) + 1))
    for left_index, left_character in enumerate(left, start=1):
        current = [left_index]
        for right_index, right_character in enumerate(right, start=1):
            current.append(
                min(
                    current[-1] + 1,
                    previous[right_index] + 1,
                    previous[right_index - 1]
                    + (left_character != right_character),
                )
            )
        previous = current
    return previous[-1]


def rank_hierarchy_paths(
    query: tuple[str, ...],
    candidates: tuple[tuple[str, ...], ...],
    *,
    level_weights: tuple[int, ...],
    max_distance: int,
    max_dp_cells: int,
) -> HierarchyPathRanking:
    if type(level_weights) is not tuple:
        raise TypeError("level_weights must be an exact tuple")
    if not _MIN_COMPARED_LEVELS <= len(level_weights) <= _MAX_LEVELS:
        raise ValueError("level weight count is outside the supported range")
    if any(
        type(weight) is not int or not 1 <= weight <= _MAX_LEVEL_WEIGHT
        for weight in level_weights
    ):
        raise ValueError("level weights must be bounded positive integers")
    if type(max_distance) is not int:
        raise TypeError("max_distance must be an integer")
    if not 0 <= max_distance <= _MAX_DISTANCE:
        raise ValueError("max_distance is outside the supported range")
    if type(max_dp_cells) is not int:
        raise TypeError("max_dp_cells must be an integer")
    if not 1 <= max_dp_cells <= _HARD_MAX_DP_CELLS:
        raise ValueError("max_dp_cells is outside the supported range")

    full_depth = len(level_weights)
    validated_query = _validate_path(query, name="query")
    if len(validated_query) not in (full_depth, full_depth - 1):
        raise ValueError(
            "query depth must equal candidate depth or be shorter by one"
        )
    if type(candidates) is not tuple:
        raise TypeError("candidates must be an exact tuple")
    if not 1 <= len(candidates) <= _MAX_CANDIDATES:
        raise ValueError("candidate count is outside the supported range")

    validated_candidates: list[tuple[str, ...]] = []
    seen_candidates: set[tuple[str, ...]] = set()
    for candidate_index, candidate in enumerate(candidates):
        validated = _validate_path(
            candidate,
            name=f"candidates[{candidate_index}]",
            expected_depth=full_depth,
        )
        if validated in seen_candidates:
            raise ValueError("candidate paths must be unique")
        seen_candidates.add(validated)
        validated_candidates.append(validated)

    if len(validated_query) == full_depth:
        options = ((PathAlignment.DIRECT, validated_query),)
    else:
        options = (
            (
                PathAlignment.QUERY_MISSING_LEADING,
                (None, *validated_query),
            ),
            (
                PathAlignment.QUERY_MISSING_TRAILING,
                (*validated_query, None),
            ),
        )

    required_cells = 0
    for candidate in validated_candidates:
        for _alignment, aligned_query in options:
            required_cells += sum(
                (len(query_level) + 1) * (len(candidate_level) + 1)
                for query_level, candidate_level in zip(
                    aligned_query,
                    candidate,
                    strict=True,
                )
                if query_level is not None
            )
    if required_cells > max_dp_cells:
        raise ValueError("edit-distance work exceeds max_dp_cells")

    scored: list[HierarchyPathScore] = []
    for candidate_index, candidate in enumerate(validated_candidates):
        option_scores: list[tuple[PathAlignment, int]] = []
        for alignment, aligned_query in options:
            distance = sum(
                weight * _levenshtein(query_level, candidate_level)
                for weight, query_level, candidate_level in zip(
                    level_weights,
                    aligned_query,
                    candidate,
                    strict=True,
                )
                if query_level is not None
            )
            option_scores.append((alignment, distance))

        candidate_distance = min(distance for _, distance in option_scores)
        scored.append(
            HierarchyPathScore(
                candidate_index=candidate_index,
                candidate=candidate,
                distance=candidate_distance,
                alignments=tuple(
                    alignment
                    for alignment, distance in option_scores
                    if distance == candidate_distance
                ),
            )
        )

    eligible = tuple(
        sorted(
            (
                score
                for score in scored
                if score.distance <= max_distance
            ),
            key=lambda score: (score.distance, score.candidate_index),
        )
    )
    best_ties = (
        tuple(score for score in eligible if score.distance == eligible[0].distance)
        if eligible
        else ()
    )
    return HierarchyPathRanking(
        ranked=eligible,
        best_ties=best_ties,
        evaluated_candidates=len(validated_candidates),
        dp_cells=required_cells,
    )
```

## Example

```python
candidates = (
    ("north", "books", "fiction"),
    ("books", "fiction", "archive"),
    ("north", "book", "fiction"),
)
ranking = rank_hierarchy_paths(
    ("books", "fiction"),
    candidates,
    level_weights=(1, 3, 5),
    max_distance=3,
    max_dp_cells=1_000,
)

try:
    rank_hierarchy_paths(
        ("north", "books", "fiction"),
        (candidates[0], candidates[0]),
        level_weights=(1, 3, 5),
        max_distance=3,
        max_dp_cells=1_000,
    )
except ValueError:
    duplicate_rejected = True
else:
    duplicate_rejected = False

assert (
    tuple(
        (score.candidate_index, score.distance, score.alignments)
        for score in ranking.ranked
    ),
    tuple(score.candidate_index for score in ranking.best_ties),
    ranking.evaluated_candidates,
    ranking.dp_cells,
    duplicate_rejected,
) == (
    (
        (0, 0, (PathAlignment.QUERY_MISSING_LEADING,)),
        (1, 0, (PathAlignment.QUERY_MISSING_TRAILING,)),
        (2, 3, (PathAlignment.QUERY_MISSING_LEADING,)),
    ),
    (0, 1),
    3,
    566,
    True,
)
```

## Trade-offs and Limitations

For every permitted alignment, the algorithm evaluates one character-level
Levenshtein table per compared hierarchy level. The preflight counts all table
cells as `(left_length + 1) * (right_length + 1)` and rejects the complete call
before dynamic programming when the caller's budget would be exceeded. Each
table uses two rows and therefore `O(min(a, b))` working memory, but runtime is
still proportional to the declared cell count. Candidates, result scores, and
ties are all materialized within fixed bounds.

An omitted boundary level has no distance cost, so use the shorter-query mode
only when that omission is already trusted domain knowledge. Internal or
multiple missing levels are rejected, and the two-level minimum prevents a
leaf-only fallback. Exact code-point comparison makes composed and decomposed
spellings, letter case, punctuation, and whitespace distinct. Duplicate
candidate paths are rejected; distance ties preserve candidate input order,
and equal leading/trailing alignments remain visible in `alignments`.

The distance threshold filters rankings but does not stop evaluation early.
The result is diagnostic only: it performs no confidence calibration,
statistical matching, mutation, persistence, or automatic remapping.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Canonical Unicode Caseless Comparison Key](build-a-canonical-unicode-caseless-comparison-key.md)
- [Match Strict Mutual Nearest Neighbors with a Comparison Budget](match-strict-mutual-nearest-neighbors-with-a-comparison-budget.md)
- [Select One Record per Key with an Explicit Ranking Rule](../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md)
<!-- catalog:related:end -->
