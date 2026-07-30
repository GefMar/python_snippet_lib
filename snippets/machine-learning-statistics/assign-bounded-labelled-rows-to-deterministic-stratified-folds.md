---
title: "Assign Bounded Labelled Rows to Deterministic Stratified Folds"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - assemble-out-of-fold-scores-from-explicit-validation-splits.md
  - encode-categories-with-out-of-fold-smoothed-target-means.md
  - ../algorithms-data-structures/map-keys-with-an-immutable-consistent-hash-ring.md
---

# Assign Bounded Labelled Rows to Deterministic Stratified Folds

## Idea and Problem

Assign labelled rows to balanced validation folds without depending on input order or Python's randomized process hash.

Versioned, length-framed SHA-256 digests provide reproducible ordering for the
strata and for rows inside each stratum. Each ranked stratum is distributed
cyclically across the folds. Carrying the fold cursor between strata makes the
complete row sequence one continuous cycle, so both global counts and counts
inside any stratum differ by at most one.

The original text is an explicit secondary sort key. Even a hypothetical digest
collision therefore has deterministic behavior.

## When to Use

Use this algorithm for bounded, uniquely identified rows when exact
reproducibility and simple single-label stratification matter more than
compatibility with a particular machine-learning framework. It can supply the
explicit validation ownership expected by a later out-of-fold calculation.

Choose a grouped, entity-aware, or chronological splitter when related rows or
future information must stay together or out of training. Similar label counts
alone do not prevent leakage.

## Implementation

```python
import hashlib
import re
from collections import Counter, defaultdict
from dataclasses import dataclass

_MAX_ROWS = 10_000
_MAX_FOLDS = 64
_MAX_SEED_BYTES = 64
_IDENTIFIER = re.compile(
    r"[A-Za-z0-9][A-Za-z0-9_.:/-]{0,127}",
    re.ASCII,
).fullmatch
_STRATUM_TAG = b"stratified-fold-v1:stratum\0"
_ROW_TAG = b"stratified-fold-v1:row\0"
_CURSOR_TAG = b"stratified-fold-v1:cursor\0"


@dataclass(frozen=True, slots=True)
class LabelledRow:
    row_id: str
    stratum: str


@dataclass(frozen=True, slots=True)
class FoldAssignment:
    row_id: str
    fold: int


def _frame(value: bytes) -> bytes:
    return len(value).to_bytes(2, "big") + value


def _digest(tag: bytes, *fields: bytes) -> bytes:
    digest = hashlib.sha256(tag)
    for field in fields:
        digest.update(_frame(field))
    return digest.digest()


def _validate_identifier(name: str, value: object) -> str:
    if type(value) is not str:
        raise TypeError(f"{name} must be an exact string")
    if _IDENTIFIER(value) is None:
        raise ValueError(f"{name} is outside the supported ASCII grammar")
    return value


def assign_stratified_folds(
    rows: tuple[LabelledRow, ...],
    fold_count: int,
    *,
    seed: bytes = b"",
) -> tuple[FoldAssignment, ...]:
    if type(rows) is not tuple:
        raise TypeError("rows must be an exact tuple")
    if not 2 <= len(rows) <= _MAX_ROWS:
        raise ValueError("row count is outside the supported range")
    if type(fold_count) is not int:
        raise TypeError("fold_count must be an exact integer")
    if not 2 <= fold_count <= _MAX_FOLDS or fold_count > len(rows):
        raise ValueError("fold_count is outside the supported range")
    if type(seed) is not bytes:
        raise TypeError("seed must be exact bytes")
    if len(seed) > _MAX_SEED_BYTES:
        raise ValueError("seed exceeds the supported byte limit")

    grouped: dict[str, list[tuple[int, LabelledRow]]] = defaultdict(list)
    seen_row_ids: set[str] = set()
    for index, row in enumerate(rows):
        if type(row) is not LabelledRow:
            raise TypeError("rows must contain exact LabelledRow records")
        row_id = _validate_identifier("row_id", row.row_id)
        stratum = _validate_identifier("stratum", row.stratum)
        if row_id in seen_row_ids:
            raise ValueError("row identifiers must be globally unique")
        seen_row_ids.add(row_id)
        grouped[stratum].append((index, row))

    stratum_order = sorted(
        grouped,
        key=lambda stratum: (
            _digest(_STRATUM_TAG, seed, stratum.encode("ascii")),
            stratum,
        ),
    )
    cursor = int.from_bytes(_digest(_CURSOR_TAG, seed), "big") % fold_count
    fold_by_index = [-1] * len(rows)

    for stratum in stratum_order:
        stratum_bytes = stratum.encode("ascii")
        ranked = sorted(
            grouped[stratum],
            key=lambda indexed_row: (
                _digest(
                    _ROW_TAG,
                    seed,
                    stratum_bytes,
                    indexed_row[1].row_id.encode("ascii"),
                ),
                indexed_row[1].row_id,
            ),
        )
        for offset, (index, _) in enumerate(ranked):
            fold_by_index[index] = (cursor + offset) % fold_count
        cursor = (cursor + len(ranked)) % fold_count

    return tuple(
        FoldAssignment(row.row_id, fold_by_index[index])
        for index, row in enumerate(rows)
    )
```

## Example

```python
rows = (
    LabelledRow("row-01", "positive"),
    LabelledRow("row-02", "negative"),
    LabelledRow("row-03", "positive"),
    LabelledRow("row-04", "negative"),
    LabelledRow("row-05", "positive"),
    LabelledRow("row-06", "negative"),
    LabelledRow("row-07", "positive"),
)

assignments = assign_stratified_folds(rows, 3, seed=b"review-split")
reversed_assignments = assign_stratified_folds(
    tuple(reversed(rows)),
    3,
    seed=b"review-split",
)

by_id = {assignment.row_id: assignment.fold for assignment in assignments}
assert by_id == {
    assignment.row_id: assignment.fold
    for assignment in reversed_assignments
}
assert tuple(assignment.row_id for assignment in assignments) == tuple(
    row.row_id for row in rows
)
assert set(by_id.values()) == {0, 1, 2}

for stratum in {row.stratum for row in rows}:
    counts = Counter(
        by_id[row.row_id] for row in rows if row.stratum == stratum
    )
    complete_counts = [counts[fold] for fold in range(3)]
    assert max(complete_counts) - min(complete_counts) <= 1

global_counts = Counter(by_id.values())
assert max(global_counts.values()) - min(global_counts.values()) <= 1
```

## Trade-offs and Limitations

For `n` rows, sorting uses `O(n log n)` time in the worst case and grouping,
digests, assignments, and output use `O(n)` space. The carried cyclic cursor
means every fold is non-empty when `fold_count <= n`; across the complete input
and within each stratum, maximum and minimum fold counts differ by at most one.

SHA-256 is used as a stable ranking primitive, not as proof of statistical
randomness or independence. A known seed permits anyone to reproduce the
partition. Changing any row, stratum, seed, fold count, framing version, or
ordering domain may reassign multiple existing rows; this is not an
incrementally stable partitioner.

Stratification preserves only bounded single-label counts. It does not prevent
group, user, temporal, spatial, or feature leakage; optimize multilabel
balance; enforce a minimum class size; train a model; or score a fold. Decide
those study-design constraints before treating the result as a valid
evaluation split.

## Related Snippets

<!-- catalog:related:start -->
- [Assemble Out-of-Fold Scores from Explicit Validation Splits](assemble-out-of-fold-scores-from-explicit-validation-splits.md)
- [Encode Categories with Out-of-Fold Smoothed Target Means](encode-categories-with-out-of-fold-smoothed-target-means.md)
- [Map Keys with an Immutable Consistent Hash Ring](../algorithms-data-structures/map-keys-with-an-immutable-consistent-hash-ring.md)
<!-- catalog:related:end -->
