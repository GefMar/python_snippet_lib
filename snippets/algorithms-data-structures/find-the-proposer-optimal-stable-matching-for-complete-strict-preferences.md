---
title: "Find the Proposer-Optimal Stable Matching for Complete Strict Preferences"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-a-deterministic-maximum-cardinality-matching-in-a-bounded-bipartite-graph.md
  - build-and-evaluate-a-bounded-binary-assignment-constraint-system.md
  - match-strict-mutual-nearest-neighbors-with-a-comparison-budget.md
---

# Find the Proposer-Optimal Stable Matching for Complete Strict Preferences

## Idea and Problem

Match two equally sized indexed sides under complete strict preferences while giving every proposer its best partner attainable in any stable matching.

Every proposer ranks every receiver, and every receiver ranks every proposer.
A matched result is stable when no proposer and receiver both prefer each other
to their assigned partners. Deferred acceptance preserves tentative receiver
choices while rejected proposers continue down their own preference rows. With
complete strict lists, the process ends in a perfect stable matching that is
optimal for every member of the proposing side among all stable matchings.

## When to Use

Use this algorithm when both sides are closed index sets of the same size,
every participant supplies a strict permutation of the other side, and the
required outcome is specifically the proposer-optimal stable matching. It fits
small, fully materialized allocation problems where stability is the governing
criterion and input validation must precede every matching decision.

The two matrices must be exact tuples with one exact tuple row per participant.
For side size `N`, each row must contain every exact integer in `range(N)` once.
The empty two-sided profile is valid. The combined input is deliberately capped
at 100,000 preference cells, so the largest accepted side has 223 participants.

## Implementation

```python
from collections import deque

_MAX_PREFERENCE_CELLS = 100_000


def _validate_complete_preference_matrix(
    matrix: tuple[tuple[int, ...], ...],
    *,
    matrix_name: str,
    size: int,
) -> None:
    if len(matrix) != size:
        raise ValueError(f"{matrix_name} must contain exactly {size} rows")

    for row_index, row in enumerate(matrix):
        if type(row) is not tuple:
            raise TypeError(f"{matrix_name}[{row_index}] must be an exact tuple")
        if len(row) != size:
            raise ValueError(f"{matrix_name}[{row_index}] must contain exactly {size} indexes")

        seen = [False] * size
        for column_index, participant in enumerate(row):
            if type(participant) is not int:
                raise TypeError(
                    f"{matrix_name}[{row_index}][{column_index}] must be an exact integer"
                )
            if not 0 <= participant < size:
                raise ValueError(f"{matrix_name}[{row_index}][{column_index}] is outside the side")
            if seen[participant]:
                raise ValueError(f"{matrix_name}[{row_index}] repeats index {participant}")
            seen[participant] = True


def stable_match_complete_preferences(
    proposer_preferences: tuple[tuple[int, ...], ...],
    receiver_preferences: tuple[tuple[int, ...], ...],
) -> tuple[int, ...]:
    """Return the receiver assigned to each proposer in proposer order."""
    if type(proposer_preferences) is not tuple:
        raise TypeError("proposer_preferences must be an exact tuple")
    if type(receiver_preferences) is not tuple:
        raise TypeError("receiver_preferences must be an exact tuple")

    size = len(proposer_preferences)
    if len(receiver_preferences) != size:
        raise ValueError("the two preference matrices must have equal side sizes")
    if 2 * size * size > _MAX_PREFERENCE_CELLS:
        raise ValueError("the combined preference-cell limit is exceeded")

    _validate_complete_preference_matrix(
        proposer_preferences,
        matrix_name="proposer_preferences",
        size=size,
    )
    _validate_complete_preference_matrix(
        receiver_preferences,
        matrix_name="receiver_preferences",
        size=size,
    )

    receiver_ranks = [[0] * size for _ in range(size)]
    for receiver, preference_row in enumerate(receiver_preferences):
        for rank, proposer in enumerate(preference_row):
            receiver_ranks[receiver][proposer] = rank

    free_proposers = deque(range(size))
    next_choice = [0] * size
    proposer_by_receiver = [-1] * size

    while free_proposers:
        proposer = free_proposers.popleft()
        receiver = proposer_preferences[proposer][next_choice[proposer]]
        next_choice[proposer] += 1

        incumbent = proposer_by_receiver[receiver]
        if incumbent == -1:
            proposer_by_receiver[receiver] = proposer
        elif receiver_ranks[receiver][proposer] < receiver_ranks[receiver][incumbent]:
            proposer_by_receiver[receiver] = proposer
            free_proposers.append(incumbent)
        else:
            free_proposers.append(proposer)

    receiver_by_proposer = [-1] * size
    for receiver, proposer in enumerate(proposer_by_receiver):
        receiver_by_proposer[proposer] = receiver
    return tuple(receiver_by_proposer)
```

## Example

```python
def exercise_complete_preference_matching() -> tuple[int, int, tuple[int, ...]]:
    from itertools import permutations, product
    from random import Random

    def is_stable_independently(
        proposer_preferences: tuple[tuple[int, ...], ...],
        receiver_preferences: tuple[tuple[int, ...], ...],
        matching: tuple[int, ...],
    ) -> bool:
        size = len(proposer_preferences)
        if tuple(sorted(matching)) != tuple(range(size)):
            return False

        proposer_by_receiver = [-1] * size
        for proposer, receiver in enumerate(matching):
            proposer_by_receiver[receiver] = proposer

        receiver_ranks = [
            {proposer: rank for rank, proposer in enumerate(preference_row)}
            for preference_row in receiver_preferences
        ]
        for proposer, preference_row in enumerate(proposer_preferences):
            assigned_receiver = matching[proposer]
            for preferred_receiver in preference_row:
                if preferred_receiver == assigned_receiver:
                    break
                incumbent = proposer_by_receiver[preferred_receiver]
                if (
                    receiver_ranks[preferred_receiver][proposer]
                    < receiver_ranks[preferred_receiver][incumbent]
                ):
                    return False
        return True

    def assert_stable_and_proposer_optimal(
        proposer_preferences: tuple[tuple[int, ...], ...],
        receiver_preferences: tuple[tuple[int, ...], ...],
    ) -> tuple[int, ...]:
        size = len(proposer_preferences)
        result = stable_match_complete_preferences(
            proposer_preferences,
            receiver_preferences,
        )
        stable_matchings = tuple(
            candidate
            for candidate in permutations(range(size))
            if is_stable_independently(
                proposer_preferences,
                receiver_preferences,
                candidate,
            )
        )

        assert result in stable_matchings
        for alternative in stable_matchings:
            for proposer, preference_row in enumerate(proposer_preferences):
                assert preference_row.index(result[proposer]) <= preference_row.index(
                    alternative[proposer]
                )
        return result

    def shuffled_preference_matrix(
        size: int,
        generator: Random,
    ) -> tuple[tuple[int, ...], ...]:
        rows = []
        for _ in range(size):
            row = list(range(size))
            generator.shuffle(row)
            rows.append(tuple(row))
        return tuple(rows)

    exhaustive_profile_count = 0
    for size in range(4):
        possible_rows = tuple(permutations(range(size)))
        possible_matrices = tuple(product(possible_rows, repeat=size))
        for proposer_matrix, receiver_matrix in product(possible_matrices, repeat=2):
            assert_stable_and_proposer_optimal(proposer_matrix, receiver_matrix)
            exhaustive_profile_count += 1

    assert exhaustive_profile_count == 46_674

    generator = Random(20_260_729)
    seeded_profile_count = 0
    for size in (4, 5):
        for _ in range(12):
            proposer_matrix = shuffled_preference_matrix(size, generator)
            receiver_matrix = shuffled_preference_matrix(size, generator)
            assert_stable_and_proposer_optimal(proposer_matrix, receiver_matrix)
            seeded_profile_count += 1

    boundary_size = 223
    cyclic_proposers = tuple(
        tuple((proposer + offset) % boundary_size for offset in range(boundary_size))
        for proposer in range(boundary_size)
    )
    cyclic_receivers = tuple(
        tuple((receiver - offset) % boundary_size for offset in range(boundary_size))
        for receiver in range(boundary_size)
    )
    cyclic_result = stable_match_complete_preferences(cyclic_proposers, cyclic_receivers)
    assert cyclic_result == tuple(range(boundary_size))
    assert is_stable_independently(cyclic_proposers, cyclic_receivers, cyclic_result)

    ascending = tuple(range(boundary_size))
    descending = tuple(reversed(ascending))
    adversarial_proposers = tuple(ascending for _ in range(boundary_size))
    adversarial_receivers = tuple(descending for _ in range(boundary_size))
    adversarial_result = stable_match_complete_preferences(
        adversarial_proposers,
        adversarial_receivers,
    )
    assert adversarial_result == descending
    assert is_stable_independently(
        adversarial_proposers,
        adversarial_receivers,
        adversarial_result,
    )

    def assert_rejected(
        expected_exception: type[Exception],
        proposer_matrix: object,
        receiver_matrix: object,
    ) -> None:
        try:
            stable_match_complete_preferences(proposer_matrix, receiver_matrix)
        except expected_exception:
            return
        raise AssertionError("invalid preferences were accepted")

    assert_rejected(TypeError, ((0, True), (1, 0)), ((0, 1), (1, 0)))
    assert_rejected(TypeError, ((0, 1), (1, 0)), ((0, 1), [1, 0]))
    assert_rejected(ValueError, ((0, 0), (1, 0)), ((0, 1), (1, 0)))
    assert_rejected(ValueError, ((0,),), ())

    return exhaustive_profile_count, seeded_profile_count, adversarial_result


exhaustive_count, seeded_count, boundary_result = exercise_complete_preference_matching()

assert (
    exhaustive_count == 46_674
    and seeded_count == 24
    and boundary_result == tuple(reversed(range(223)))
)
```

## Trade-offs and Limitations

Validation, receiver-rank construction, and deferred acceptance take `O(N^2)`
time. Each proposer approaches each receiver at most once. Receiver ranks use
`O(N^2)` auxiliary slots; the queue, partner arrays, and next-choice indexes
use another `O(N)`. The condition `2 * N * N <= 100_000` deliberately limits
the implementation to complete matrices with at most 223 participants per
side.

The result is stable, perfect, and proposer-optimal for complete strict
preferences. Proposer optimality means every proposer weakly prefers this
partner to its partner in any other stable matching. Receivers can prefer a
different stable matching, and exchanging the proposing and receiving sides
can change the result. Stability alone does not optimize total welfare,
fairness, distance, weight, or any cross-participant score.

This function does not support ties, incomplete lists, unacceptable partners,
unequal side sizes, capacities, weighted objectives, or dynamic updates. Those
variants need a different contract and sometimes a different algorithm. It
also makes no strategy-proofness claim outside the stated one-sided complete
strict-preference model.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Deterministic Maximum-Cardinality Matching in a Bounded Bipartite Graph](find-a-deterministic-maximum-cardinality-matching-in-a-bounded-bipartite-graph.md)
- [Build and Evaluate a Bounded Binary Assignment Constraint System](build-and-evaluate-a-bounded-binary-assignment-constraint-system.md)
- [Match Strict Mutual Nearest Neighbors with a Comparison Budget](match-strict-mutual-nearest-neighbors-with-a-comparison-budget.md)
<!-- catalog:related:end -->
