---
title: "Choose the First Eligible Candidate from Ordered Priority Groups"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md
  - ../networking-protocols/choose-grouped-endpoints-with-explicit-random-fairness.md
  - ../concurrency-lifecycle/plan-priority-batches-with-an-age-gated-tail.md
---

# Choose the First Eligible Candidate from Ordered Priority Groups

## Idea and Problem

Choose one eligible candidate deterministically by validating every group first and then respecting group priority followed by candidate order.

A flat eligible set alone does not express fallback priority. Ordered groups
make that policy explicit, while global candidate uniqueness and a subset check
prevent a duplicated or undeclared identifier from changing which position
wins.

## When to Use

Use this algorithm when a caller already owns a complete, immutable eligibility
snapshot and group order must dominate member order. Examples include choosing
one compatible implementation tier, one processing lane, or one local fallback
option without randomness.

Keep dynamic lookups outside the function. If eligibility can change during a
decision, first freeze the relevant snapshot or use a coordinator with the
required consistency semantics. The eligibility snapshot may contain at most
512 IDs and must be a subset of the declared candidates.

## Implementation

```python
import re
from dataclasses import dataclass


_MAX_GROUPS = 16
_MAX_CANDIDATES_PER_GROUP = 64
_MAX_TOTAL_CANDIDATES = 512
_IDENTIFIER = re.compile(r"[A-Za-z][A-Za-z0-9_-]{0,63}", re.ASCII)


class NoEligibleCandidateError(LookupError):
    pass


@dataclass(frozen=True, slots=True)
class PriorityGroup:
    group_id: str
    candidate_ids: tuple[str, ...]


@dataclass(frozen=True, slots=True)
class CandidateChoice:
    group_id: str
    candidate_id: str
    group_index: int
    candidate_index: int


def _validated_identifier(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be text")
    if _IDENTIFIER.fullmatch(value) is None:
        raise ValueError(f"{field} must be a conservative ASCII identifier")
    return value


def choose_first_eligible_candidate(
    groups: tuple[PriorityGroup, ...],
    eligible_ids: frozenset[str],
) -> CandidateChoice:
    if type(groups) is not tuple:
        raise TypeError("groups must be an exact tuple")
    if not 1 <= len(groups) <= _MAX_GROUPS:
        raise ValueError(f"groups must contain between 1 and {_MAX_GROUPS} entries")
    if type(eligible_ids) is not frozenset:
        raise TypeError("eligible_ids must be an exact frozenset")
    if len(eligible_ids) > _MAX_TOTAL_CANDIDATES:
        raise ValueError(
            f"eligible_ids may contain at most {_MAX_TOTAL_CANDIDATES} entries"
        )

    group_ids: set[str] = set()
    declared_candidates: set[str] = set()
    total_candidates = 0

    for group_index, group in enumerate(groups):
        if type(group) is not PriorityGroup:
            raise TypeError(f"groups[{group_index}] must be a PriorityGroup")
        group_id = _validated_identifier(
            group.group_id,
            field=f"groups[{group_index}].group_id",
        )
        if group_id in group_ids:
            raise ValueError("group IDs must be unique")
        group_ids.add(group_id)

        if type(group.candidate_ids) is not tuple:
            raise TypeError(
                f"groups[{group_index}].candidate_ids must be an exact tuple"
            )
        if len(group.candidate_ids) > _MAX_CANDIDATES_PER_GROUP:
            raise ValueError(
                f"each group may contain at most {_MAX_CANDIDATES_PER_GROUP} candidates"
            )

        total_candidates += len(group.candidate_ids)
        if total_candidates > _MAX_TOTAL_CANDIDATES:
            raise ValueError(
                f"the groups may contain at most {_MAX_TOTAL_CANDIDATES} candidates"
            )
        for candidate_index, raw_candidate_id in enumerate(group.candidate_ids):
            candidate_id = _validated_identifier(
                raw_candidate_id,
                field=(
                    f"groups[{group_index}].candidate_ids[{candidate_index}]"
                ),
            )
            if candidate_id in declared_candidates:
                raise ValueError("candidate IDs must be globally unique")
            declared_candidates.add(candidate_id)

    if any(type(candidate_id) is not str for candidate_id in eligible_ids):
        raise TypeError("eligible candidate IDs must be exact strings")
    if any(
        _IDENTIFIER.fullmatch(candidate_id) is None
        for candidate_id in eligible_ids
    ):
        raise ValueError("eligible candidate IDs must be conservative ASCII identifiers")
    if not eligible_ids <= declared_candidates:
        raise ValueError("eligible IDs must be a subset of declared candidates")

    for group_index, group in enumerate(groups):
        for candidate_index, candidate_id in enumerate(group.candidate_ids):
            if candidate_id in eligible_ids:
                return CandidateChoice(
                    group_id=group.group_id,
                    candidate_id=candidate_id,
                    group_index=group_index,
                    candidate_index=candidate_index,
                )

    raise NoEligibleCandidateError("no declared candidate is eligible")
```

## Example

```python
groups = (
    PriorityGroup("preferred", ()),
    PriorityGroup("standard", ("option-a", "option-b")),
    PriorityGroup("reserve", ("option-c",)),
)

choice = choose_first_eligible_candidate(
    groups,
    frozenset({"option-b", "option-c"}),
)

no_match = False
try:
    choose_first_eligible_candidate(groups, frozenset())
except NoEligibleCandidateError:
    no_match = True

assert (choice, no_match) == (
    CandidateChoice(
        group_id="standard",
        candidate_id="option-b",
        group_index=1,
        candidate_index=1,
    ),
    True,
)
```

## Trade-offs and Limitations

Validation and selection are linear in at most 512 declared candidates and 512
eligible IDs and use linear auxiliary space for uniqueness checks. Every input
is validated before selection, so an invalid later group rejects the whole
request even when an earlier eligible candidate exists. Empty groups and an
empty eligibility set are valid inputs; the latter reaches the dedicated
no-match error.

Global candidate uniqueness makes the winning coordinates unambiguous but
prevents deliberately repeating one fallback in several groups. The algorithm
implements strict first-match priority, not scoring, weights, random fairness,
load balancing, or rotation, so a permanently eligible early candidate can
always win.

This pure function does not fetch eligibility, inspect identities, authorize a
choice, reserve capacity, perform an assignment, retry, synchronize concurrent
callers, or do I/O. Its frozen inputs must represent one caller-owned decision
snapshot.

## Related Snippets

<!-- catalog:related:start -->
- [Select One Record per Key with an Explicit Ranking Rule](../data-processing/select-one-record-per-key-with-an-explicit-ranking-rule.md)
- [Choose Grouped Endpoints with Explicit Random Fairness](../networking-protocols/choose-grouped-endpoints-with-explicit-random-fairness.md)
- [Plan Priority Batches with an Age-Gated Tail](../concurrency-lifecycle/plan-priority-batches-with-an-age-gated-tail.md)
<!-- catalog:related:end -->
