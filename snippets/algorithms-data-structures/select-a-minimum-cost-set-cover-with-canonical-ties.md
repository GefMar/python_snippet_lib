---
title: "Select a Minimum-Cost Set Cover with Canonical Ties"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - search-a-bounded-exact-cover-system-with-algorithm-x.md
  - choose-a-deterministic-minimum-point-set-stabbing-bounded-closed-integer-intervals.md
  - solve-a-bounded-zero-one-knapsack-with-canonical-item-ties.md
---

# Select a Minimum-Cost Set Cover with Canonical Ties

## Idea and Problem

Choose a minimum-cost subset of bounded candidate sets while resolving every optimum deterministically.

Represent each candidate's covered elements as an integer bitmask. A dynamic
program maps every reachable union mask to its best selection. The comparison
key first minimizes total cost, then the number of selected candidates, then
the lexicographic tuple of their original indices.

Taking candidates in input order and extending only a snapshot of previously
reachable states makes each candidate zero-or-one. Original indices remain a
stable tie-break even when several candidates cover the same elements.

## When to Use

Use this exact solver when the universe is small, candidate costs are explicit,
and a reproducible optimum matters more than scaling to many universe elements.
It is useful for bounded test oracles, choosing a small bundle of capabilities,
or selecting a least-cost group of checks that collectively cover every
required condition.

Use an integer-programming solver, a specialized branch-and-bound method, or a
documented approximation for larger instances. Set cover is NP-hard; bounding
the input does not turn this dynamic program, which is exponential in the
universe size, into a large-scale optimizer.

## Implementation

```python
from dataclasses import dataclass

_MAX_UNIVERSE_SIZE = 20
_MAX_CANDIDATES = 256
_MAX_COST = (1 << 63) - 1
_MAX_STATE_VISITS = 4_000_000


@dataclass(frozen=True, slots=True)
class SetCoverCandidate:
    elements: frozenset[int]
    cost: int


@dataclass(frozen=True, slots=True)
class SetCoverResult:
    total_cost: int
    selected_indices: tuple[int, ...]


def _selection_key(
    selection: tuple[int, tuple[int, ...]],
) -> tuple[int, int, tuple[int, ...]]:
    cost, indices = selection
    return cost, len(indices), indices


def minimum_cost_set_cover(
    universe_size: int,
    candidates: tuple[SetCoverCandidate, ...],
) -> SetCoverResult:
    """Return minimum cost, then count, then original-index canonical cover."""
    if type(universe_size) is not int:
        raise TypeError("universe_size must be an exact integer")
    if not 1 <= universe_size <= _MAX_UNIVERSE_SIZE:
        raise ValueError("universe_size is outside the supported range")
    if type(candidates) is not tuple:
        raise TypeError("candidates must be an exact tuple")
    if not 1 <= len(candidates) <= _MAX_CANDIDATES:
        raise ValueError("candidate count is outside the supported range")
    if len(candidates) * (1 << universe_size) > _MAX_STATE_VISITS:
        raise ValueError("worst-case DP work exceeds the supported limit")

    candidate_masks: list[int] = []
    available_mask = 0
    for candidate_index, candidate in enumerate(candidates):
        if type(candidate) is not SetCoverCandidate:
            raise TypeError(f"candidates[{candidate_index}] must be an exact SetCoverCandidate")
        if type(candidate.elements) is not frozenset:
            raise TypeError(f"candidates[{candidate_index}].elements must be an exact frozenset")
        if not candidate.elements:
            raise ValueError(f"candidates[{candidate_index}] must not be empty")

        mask = 0
        for element in candidate.elements:
            if type(element) is not int:
                raise TypeError("candidate elements must be exact integers")
            if not 0 <= element < universe_size:
                raise ValueError("candidate element is outside the universe")
            mask |= 1 << element

        if type(candidate.cost) is not int:
            raise TypeError(f"candidates[{candidate_index}].cost must be exact int")
        if not 0 <= candidate.cost <= _MAX_COST:
            raise ValueError("candidate cost is outside the signed 64-bit range")

        candidate_masks.append(mask)
        available_mask |= mask

    full_mask = (1 << universe_size) - 1
    if available_mask != full_mask:
        raise ValueError("candidates cannot cover the declared universe")

    best_by_mask: dict[int, tuple[int, tuple[int, ...]]] = {0: (0, ())}
    for candidate_index, (candidate, candidate_mask) in enumerate(
        zip(candidates, candidate_masks, strict=True)
    ):
        previous_states = tuple(best_by_mask.items())
        for covered_mask, (covered_cost, selected_indices) in previous_states:
            combined_mask = covered_mask | candidate_mask
            if combined_mask == covered_mask:
                continue
            proposed = (
                covered_cost + candidate.cost,
                (*selected_indices, candidate_index),
            )
            incumbent = best_by_mask.get(combined_mask)
            if incumbent is None or _selection_key(proposed) < _selection_key(incumbent):
                best_by_mask[combined_mask] = proposed

    result = best_by_mask.get(full_mask)
    if result is None:
        raise RuntimeError("validated coverage did not produce a complete DP state")
    total_cost, selected_indices = result
    return SetCoverResult(total_cost, selected_indices)


```

## Example

```python
equal_pair_choices = (
    SetCoverCandidate(frozenset({0, 1}), 3),
    SetCoverCandidate(frozenset({2, 3}), 3),
    SetCoverCandidate(frozenset({0, 2}), 3),
    SetCoverCandidate(frozenset({1, 3}), 3),
)
lexical_tie = minimum_cost_set_cover(4, equal_pair_choices)

fewest_sets = minimum_cost_set_cover(
    4,
    (*equal_pair_choices, SetCoverCandidate(frozenset({0, 1, 2, 3}), 6)),
)
cheapest = minimum_cost_set_cover(
    3,
    (
        SetCoverCandidate(frozenset({0, 1}), 4),
        SetCoverCandidate(frozenset({2}), 1),
        SetCoverCandidate(frozenset({0, 1, 2}), 6),
    ),
)
large_non_optimal_cost = minimum_cost_set_cover(
    1,
    (
        SetCoverCandidate(frozenset({0}), (1 << 63) - 1),
        SetCoverCandidate(frozenset({0}), 1),
    ),
)

assert (lexical_tie, fewest_sets, cheapest, large_non_optimal_cost) == (
    SetCoverResult(6, (0, 1)),
    SetCoverResult(6, (4,)),
    SetCoverResult(5, (0, 1)),
    SetCoverResult(1, (1,)),
)
```

## Trade-offs and Limitations

For `k` candidates and universe size `u`, the worst case uses
`O(k * 2**u)` state visits and `O(2**u)` dictionary entries. Witness tuples
also consume memory proportional to their selected indices. The explicit
four-million-visit check bounds the dominant loop before allocation grows.
Each candidate cost is bounded to the non-negative signed-64-bit range, while
Python's exact integers allow a selected total to exceed that range when the
only complete cover requires several expensive candidates.

Candidate order is semantically observable only after cost and selected-count
ties: the lexicographically smallest tuple of original indices wins. Reordering
otherwise identical input can therefore change the witness without changing
its optimal cost or size. Zero-cost candidates are supported, but an admitted
candidate that adds no new coverage is never selected because fewer sets win.

This solver covers every universe element at least once. It does not require
exactly-once coverage, attach weights to universe elements, support negative
costs, select a candidate repeatedly, enumerate all optima, approximate large
instances, or prove that an arbitrary real-world requirement model is
complete.

## Related Snippets

<!-- catalog:related:start -->
- [Search a Bounded Exact-Cover System with Algorithm X](search-a-bounded-exact-cover-system-with-algorithm-x.md)
- [Choose a Deterministic Minimum Point Set Stabbing Bounded Closed Integer Intervals](choose-a-deterministic-minimum-point-set-stabbing-bounded-closed-integer-intervals.md)
- [Solve a Bounded Zero-One Knapsack with Canonical Item Ties](solve-a-bounded-zero-one-knapsack-with-canonical-item-ties.md)
<!-- catalog:related:end -->
