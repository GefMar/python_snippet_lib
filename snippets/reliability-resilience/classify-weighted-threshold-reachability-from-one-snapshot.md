---
title: "Classify Weighted Threshold Reachability from One Snapshot"
snippet_type: algorithm
use_cases:
  - lifecycle-management
  - retry-recovery
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - decide-whether-a-bounded-work-snapshot-permits-a-new-attempt.md
  - ../algorithms-data-structures/find-a-strict-majority-with-boyer-moore-voting.md
  - ../observability-operations/classify-progress-from-complete-bounded-counter-snapshots.md
---

# Classify Weighted Threshold Reachability from One Snapshot

## Idea and Problem

Determine whether an aggregate weighted outcome has met a required threshold, can still meet it through unknown weight, or can no longer meet it.

The snapshot partitions its total weight into successful, failed, and unknown
outcomes. Success at or above the threshold is `MET`. Otherwise, an outcome is
`IMPOSSIBLE` when even all unknown weight becoming successful would remain
below the threshold; every other snapshot is `PENDING`.

## When to Use

Use this algorithm after obtaining one coherent, immutable aggregate from a
fixed set of weighted outcomes. It fits a bounded approval or completion gate
whose unresolved outcomes can only settle as success or failure and whose
caller needs pure advice about whether to keep waiting.

Choose the weights and required threshold outside this function. Preserve
participant identities before aggregation when membership, duplicate
submissions, authorization, or audit evidence matter. Recheck authoritative
state before acting when the underlying snapshot can change.

## Implementation

```python
from enum import StrEnum

_MAX_INT64 = (1 << 63) - 1


class ThresholdReachability(StrEnum):
    MET = "met"
    PENDING = "pending"
    IMPOSSIBLE = "impossible"


def _validated_weight(value: object, *, name: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact non-boolean integer")
    if not 0 <= value <= _MAX_INT64:
        raise ValueError(f"{name} is outside the supported range")
    return value


def classify_weighted_threshold_reachability(
    *,
    success_weight: int,
    failure_weight: int,
    unknown_weight: int,
    required_weight: int,
) -> ThresholdReachability:
    """Classify threshold reachability from one aggregate snapshot."""
    success = _validated_weight(success_weight, name="success_weight")
    failure = _validated_weight(failure_weight, name="failure_weight")
    unknown = _validated_weight(unknown_weight, name="unknown_weight")

    total_weight = success + failure + unknown
    if not 1 <= total_weight <= _MAX_INT64:
        raise ValueError("aggregate weight is outside the supported range")

    if type(required_weight) is not int:
        raise TypeError("required_weight must be an exact non-boolean integer")
    if not 1 <= required_weight <= total_weight:
        raise ValueError("required_weight is outside the aggregate weight range")

    if success >= required_weight:
        return ThresholdReachability.MET
    if success + unknown < required_weight:
        return ThresholdReachability.IMPOSSIBLE
    return ThresholdReachability.PENDING
```

## Example

```python
met = classify_weighted_threshold_reachability(
    success_weight=6,
    failure_weight=3,
    unknown_weight=1,
    required_weight=6,
)
pending_at_boundary = classify_weighted_threshold_reachability(
    success_weight=4,
    failure_weight=3,
    unknown_weight=2,
    required_weight=6,
)
impossible = classify_weighted_threshold_reachability(
    success_weight=4,
    failure_weight=4,
    unknown_weight=1,
    required_weight=6,
)

try:
    classify_weighted_threshold_reachability(
        success_weight=True,
        failure_weight=0,
        unknown_weight=0,
        required_weight=1,
    )
except TypeError:
    bool_rejected = True
else:
    bool_rejected = False

assert (met, pending_at_boundary, impossible, bool_rejected) == (
    ThresholdReachability.MET,
    ThresholdReachability.PENDING,
    ThresholdReachability.IMPOSSIBLE,
    True,
)
```

## Trade-offs and Limitations

Validation and classification use `O(1)` time and auxiliary memory. Every
individual weight is a non-negative signed-64-bit integer, and their positive
aggregate must also fit that range. Exact type checks deliberately reject
Booleans even though `bool` is an `int` subclass in Python.

`MET` remains terminal only when recorded successes cannot be revoked.
`IMPOSSIBLE` remains terminal only when failures are final and unknown weight
is the only remaining source of success. Equality of success plus unknown with
the threshold is still `PENDING`, because all unknown weight could satisfy the
requirement.

The function does not acquire a snapshot, identify or deduplicate voters,
authenticate inputs, choose weights, persist decisions, manage changing
membership, or establish quorum intersection, Byzantine safety, or distributed
consensus. It classifies one already aggregated snapshot and performs no action.

## Related Snippets

<!-- catalog:related:start -->
- [Decide Whether a Bounded Work Snapshot Permits a New Attempt](decide-whether-a-bounded-work-snapshot-permits-a-new-attempt.md)
- [Find a Strict Majority with Boyer-Moore Voting](../algorithms-data-structures/find-a-strict-majority-with-boyer-moore-voting.md)
- [Classify Progress from Complete Bounded Counter Snapshots](../observability-operations/classify-progress-from-complete-bounded-counter-snapshots.md)
<!-- catalog:related:end -->
