---
title: "Plan Gap-Separated Expanding-Window Validation Splits for Ordered Rows"
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
  - assign-bounded-labelled-rows-to-deterministic-stratified-folds.md
  - encode-categories-with-out-of-fold-smoothed-target-means.md
---

# Plan Gap-Separated Expanding-Window Validation Splits for Ordered Rows

## Idea and Problem

Plan bounded expanding training ranges followed by a caller-sized gap and one fixed-width validation range.

A cursor begins after the initial training prefix. Every complete split uses
that cursor as its training stop, skips the current gap, and admits a
validation range only when the whole range fits. Moving the cursor to the
validation stop makes the next training prefix absorb all earlier rows while
preserving a fresh gap before its validation rows.

## When to Use

Use this planner when row order is already authoritative and a walk-forward
evaluation needs reproducible integer boundaries without slicing data or
depending on a machine-learning framework. The explicit unused tail and
omission flag make partial final windows and a restrictive split cap visible.

The gap is only a positional policy supplied by the caller. Use a splitter
that understands timestamps, label horizons, entities, or groups when those
relationships determine leakage. Use rolling training windows when old rows
must expire instead of remaining in every later training prefix.

## Implementation

```python
from dataclasses import dataclass

_MAX_ROWS = 1_000_000
_MAX_SPLITS = 128


@dataclass(frozen=True, slots=True)
class RowRange:
    start: int
    stop: int


@dataclass(frozen=True, slots=True)
class ValidationSplit:
    training: RowRange
    gap: RowRange
    validation: RowRange


@dataclass(frozen=True, slots=True)
class ExpandingWindowPlan:
    splits: tuple[ValidationSplit, ...]
    unused_tail: RowRange
    has_omitted_complete_split: bool


def plan_gap_separated_splits(
    row_count: int,
    *,
    initial_training_size: int,
    validation_size: int,
    gap_size: int,
    split_cap: int,
) -> ExpandingWindowPlan:
    values = (
        ("row_count", row_count),
        ("initial_training_size", initial_training_size),
        ("validation_size", validation_size),
        ("gap_size", gap_size),
        ("split_cap", split_cap),
    )
    for name, value in values:
        if type(value) is not int:
            raise TypeError(f"{name} must be an exact integer")

    if not 1 <= row_count <= _MAX_ROWS:
        raise ValueError("row_count is outside the supported range")
    if not 1 <= initial_training_size <= row_count:
        raise ValueError("initial_training_size is outside the row range")
    if not 1 <= validation_size <= row_count:
        raise ValueError("validation_size is outside the row range")
    if not 0 <= gap_size <= row_count:
        raise ValueError("gap_size is outside the row range")
    if not 1 <= split_cap <= _MAX_SPLITS:
        raise ValueError("split_cap is outside the supported range")

    cursor = initial_training_size
    splits: list[ValidationSplit] = []
    while len(splits) < split_cap:
        validation_start = cursor + gap_size
        validation_stop = validation_start + validation_size
        if validation_stop > row_count:
            break
        splits.append(
            ValidationSplit(
                training=RowRange(0, cursor),
                gap=RowRange(cursor, validation_start),
                validation=RowRange(validation_start, validation_stop),
            )
        )
        cursor = validation_stop

    if not splits:
        raise ValueError("configuration does not contain a complete split")

    has_omitted_complete_split = (
        len(splits) == split_cap and cursor + gap_size + validation_size <= row_count
    )
    return ExpandingWindowPlan(
        splits=tuple(splits),
        unused_tail=RowRange(cursor, row_count),
        has_omitted_complete_split=has_omitted_complete_split,
    )


```

## Example

```python
capped = plan_gap_separated_splits(
    31,
    initial_training_size=5,
    validation_size=4,
    gap_size=2,
    split_cap=3,
)

assert tuple((split.training, split.gap, split.validation) for split in capped.splits) == (
    (RowRange(0, 5), RowRange(5, 7), RowRange(7, 11)),
    (RowRange(0, 11), RowRange(11, 13), RowRange(13, 17)),
    (RowRange(0, 17), RowRange(17, 19), RowRange(19, 23)),
)
assert capped.unused_tail == RowRange(23, 31)
assert capped.has_omitted_complete_split

complete = plan_gap_separated_splits(
    31,
    initial_training_size=5,
    validation_size=4,
    gap_size=2,
    split_cap=10,
)
assert (len(complete.splits), complete.unused_tail) == (4, RowRange(29, 31))
assert not complete.has_omitted_complete_split
```

## Trade-offs and Limitations

Planning `s` complete splits costs `O(s)` time and memory, with `s` capped at
128. Returned ranges are frozen half-open coordinates; they neither copy nor
retain the source rows. A zero-sized gap is represented explicitly as an
empty `[cursor, cursor)` range.

Every later training range begins at zero and ends at the previous validation
stop, so earlier gap and validation rows eventually join training. The unused
tail is only the suffix after the last emitted validation. The omission flag
is true only when the split cap hides another whole gap-plus-validation block,
not when the tail is merely too short.

Exact built-in integers reject booleans and numeric subclasses. The planner
does not inspect data, shuffle rows, train a model, create rolling windows, or
prove that its positional gap prevents temporal, group, or label leakage.

## Related Snippets

<!-- catalog:related:start -->
- [Assemble Out-of-Fold Scores from Explicit Validation Splits](assemble-out-of-fold-scores-from-explicit-validation-splits.md)
- [Assign Bounded Labelled Rows to Deterministic Stratified Folds](assign-bounded-labelled-rows-to-deterministic-stratified-folds.md)
- [Encode Categories with Out-of-Fold Smoothed Target Means](encode-categories-with-out-of-fold-smoothed-target-means.md)
<!-- catalog:related:end -->
