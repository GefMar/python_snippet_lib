---
title: "Find a Canonical Minimum-Cost State Path in a Bounded Trellis with Viterbi Dynamic Programming"
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
  - align-bounded-integer-sequences-with-exact-dynamic-time-warping.md
  - find-a-deterministic-critical-path-in-a-bounded-vertex-weighted-dag.md
  - find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md
---

# Find a Canonical Minimum-Cost State Path in a Bounded Trellis with Viterbi Dynamic Programming

## Idea and Problem

Find the least-cost state sequence through a dense, time-layered trellis while making every equal-cost result canonical.

At time zero, a path pays its state's initial cost and first emission cost.
Every later layer adds the transition from the preceding state and the current
emission. `None` forbids an initial state, transition, or emission.

Dynamic programming keeps only the best prefix ending at each state. To select
the lexicographically smallest complete state tuple without copying paths in
the inner loop, each reachable prefix receives its lexicographic rank. A
predecessor is selected by `(cost, prefix rank)`, and the next ranks are the
sorted order of `(predecessor rank, current state)`. Backpointers retain enough
information to reconstruct the winning path after the final layer.

## When to Use

Use this for a complete, bounded trellis with exact non-negative integer costs,
such as a small offline decoder or a deterministic reference implementation.
State indexes define the tie policy, so their order must be stable and
meaningful to the caller.

The inputs must be exact immutable tuples. There may be 1..64 states and
1..256 time steps, with at most 2,000,000 `time * states**2` transition checks.
Every present local cost must be an exact integer in `0..1_000_000`.

Use a sparse trellis implementation when most transitions are forbidden. Use
a domain-specific probabilistic implementation when probability conversion,
model training, posterior probabilities, or numerical scaling are part of the
problem rather than already-resolved integer costs.

## Implementation

```python
from dataclasses import dataclass

_MAX_VITERBI_STATES = 64
_MAX_VITERBI_STEPS = 256
_MAX_VITERBI_LOCAL_COST = 1_000_000
_MAX_VITERBI_WORK = 2_000_000


@dataclass(frozen=True, slots=True)
class ViterbiResult:
    total_cost: int
    states: tuple[int, ...]


def _validate_viterbi_cost(value: object, *, field: str) -> int | None:
    if value is None:
        return None
    if type(value) is not int:
        raise TypeError(f"{field} must be None or an exact integer")
    if not 0 <= value <= _MAX_VITERBI_LOCAL_COST:
        raise ValueError(f"{field} is outside 0..1_000_000")
    return value


def find_minimum_cost_state_path(
    initial_costs: tuple[int | None, ...],
    transition_costs: tuple[tuple[int | None, ...], ...],
    emission_costs: tuple[tuple[int | None, ...], ...],
) -> ViterbiResult | None:
    """Return the least-cost path, breaking ties by the full state tuple."""
    if type(initial_costs) is not tuple:
        raise TypeError("initial_costs must be an exact tuple")
    state_count = len(initial_costs)
    if not 1 <= state_count <= _MAX_VITERBI_STATES:
        raise ValueError("initial_costs must contain 1..64 states")

    if type(transition_costs) is not tuple:
        raise TypeError("transition_costs must be an exact tuple")
    if len(transition_costs) != state_count:
        raise ValueError("transition_costs must have one row per state")

    if type(emission_costs) is not tuple:
        raise TypeError("emission_costs must be an exact tuple")
    step_count = len(emission_costs)
    if step_count < 1:
        raise ValueError("emission_costs must contain at least one time step")
    if step_count * state_count * state_count > _MAX_VITERBI_WORK:
        raise ValueError("trellis transition work exceeds the supported limit")
    if step_count > _MAX_VITERBI_STEPS:
        raise ValueError("emission_costs must contain at most 256 time steps")

    checked_initial = tuple(
        _validate_viterbi_cost(value, field=f"initial_costs[{state}]")
        for state, value in enumerate(initial_costs)
    )

    checked_transitions: list[tuple[int | None, ...]] = []
    for source, row in enumerate(transition_costs):
        if type(row) is not tuple:
            raise TypeError(f"transition_costs[{source}] must be an exact tuple")
        if len(row) != state_count:
            raise ValueError("every transition_costs row must have one value per state")
        checked_transitions.append(
            tuple(
                _validate_viterbi_cost(
                    value,
                    field=f"transition_costs[{source}][{target}]",
                )
                for target, value in enumerate(row)
            )
        )

    checked_emissions: list[tuple[int | None, ...]] = []
    for step, row in enumerate(emission_costs):
        if type(row) is not tuple:
            raise TypeError(f"emission_costs[{step}] must be an exact tuple")
        if len(row) != state_count:
            raise ValueError("every emission_costs row must have one value per state")
        checked_emissions.append(
            tuple(
                _validate_viterbi_cost(
                    value,
                    field=f"emission_costs[{step}][{state}]",
                )
                for state, value in enumerate(row)
            )
        )

    costs: list[int | None] = [None] * state_count
    ranks = [-1] * state_count
    next_rank = 0
    for state in range(state_count):
        initial = checked_initial[state]
        emission = checked_emissions[0][state]
        if initial is not None and emission is not None:
            costs[state] = initial + emission
            ranks[state] = next_rank
            next_rank += 1

    if next_rank == 0:
        return None

    parents: list[list[int]] = [[-1] * state_count]
    for step in range(1, step_count):
        following_costs: list[int | None] = [None] * state_count
        following_parents = [-1] * state_count

        for state in range(state_count):
            emission = checked_emissions[step][state]
            if emission is None:
                continue

            best_key: tuple[int, int] | None = None
            best_predecessor = -1
            for predecessor in range(state_count):
                prefix_cost = costs[predecessor]
                transition = checked_transitions[predecessor][state]
                if prefix_cost is None or transition is None:
                    continue
                candidate_key = (prefix_cost + transition + emission, ranks[predecessor])
                if best_key is None or candidate_key < best_key:
                    best_key = candidate_key
                    best_predecessor = predecessor

            if best_key is not None:
                following_costs[state] = best_key[0]
                following_parents[state] = best_predecessor

        reachable_states = [
            state for state, total_cost in enumerate(following_costs) if total_cost is not None
        ]
        if not reachable_states:
            return None

        reachable_states.sort(key=lambda state: (ranks[following_parents[state]], state))
        following_ranks = [-1] * state_count
        for rank, state in enumerate(reachable_states):
            following_ranks[state] = rank

        parents.append(following_parents)
        costs = following_costs
        ranks = following_ranks

    final_state = min(
        (state for state, total_cost in enumerate(costs) if total_cost is not None),
        key=lambda state: (costs[state], ranks[state]),
    )
    total_cost = costs[final_state]
    if total_cost is None:
        raise AssertionError("the selected final state must be reachable")

    reversed_states = [final_state]
    state = final_state
    for step in range(step_count - 1, 0, -1):
        state = parents[step][state]
        if state < 0:
            raise AssertionError("a reachable state must have a predecessor")
        reversed_states.append(state)

    return ViterbiResult(total_cost, tuple(reversed(reversed_states)))
```

## Example

```python
from itertools import product
from random import Random


def brute_force_state_path(
    initial_costs: tuple[int | None, ...],
    transition_costs: tuple[tuple[int | None, ...], ...],
    emission_costs: tuple[tuple[int | None, ...], ...],
) -> ViterbiResult | None:
    """Independent tiny-input oracle that enumerates every complete path."""
    best: ViterbiResult | None = None
    state_count = len(initial_costs)
    for states in product(range(state_count), repeat=len(emission_costs)):
        initial = initial_costs[states[0]]
        first_emission = emission_costs[0][states[0]]
        if initial is None or first_emission is None:
            continue
        total = initial + first_emission

        allowed = True
        for step in range(1, len(emission_costs)):
            transition = transition_costs[states[step - 1]][states[step]]
            emission = emission_costs[step][states[step]]
            if transition is None or emission is None:
                allowed = False
                break
            total += transition + emission
        if not allowed:
            continue

        candidate = ViterbiResult(total, states)
        if best is None or (candidate.total_cost, candidate.states) < (
            best.total_cost,
            best.states,
        ):
            best = candidate
    return best


all_zero_tie = find_minimum_cost_state_path(
    (0, 0),
    ((0, 0), (0, 0)),
    ((0, 0), (0, 0), (0, 0), (0, 0)),
)
one_state = find_minimum_cost_state_path(
    (2,),
    ((3,),),
    ((5,), (7,), (11,)),
)
unreachable = find_minimum_cost_state_path(
    (0, None),
    ((None, None), (None, None)),
    ((0, 0), (0, 0)),
)


def rejects(
    initial_costs: object,
    transition_costs: object,
    emission_costs: object,
) -> bool:
    try:
        find_minimum_cost_state_path(
            initial_costs,  # type: ignore[arg-type]
            transition_costs,  # type: ignore[arg-type]
            emission_costs,  # type: ignore[arg-type]
        )
    except (TypeError, ValueError):
        return True
    return False


wide_row = (None,) * 64
cap_rejected = rejects(
    (0,) * 64,
    (wide_row,) * 64,
    (wide_row,) * 489,
)

generator = Random(25_071)
checked_cases = 0
local_costs = (None, 0, 1, 2, 3)
for _ in range(512):
    small_state_count = generator.randint(1, 4)
    small_step_count = generator.randint(1, 5)
    small_initial = tuple(generator.choice(local_costs) for _ in range(small_state_count))
    small_transitions = tuple(
        tuple(generator.choice(local_costs) for _ in range(small_state_count))
        for _ in range(small_state_count)
    )
    small_emissions = tuple(
        tuple(generator.choice(local_costs) for _ in range(small_state_count))
        for _ in range(small_step_count)
    )
    assert find_minimum_cost_state_path(
        small_initial,
        small_transitions,
        small_emissions,
    ) == brute_force_state_path(
        small_initial,
        small_transitions,
        small_emissions,
    )
    checked_cases += 1

assert (
    all_zero_tie,
    one_state,
    unreachable,
    checked_cases,
    rejects([0], ((0,),), ((0,),)),
    rejects((0, 0), ((0,),), ((0, 0),)),
    rejects((0,), ((0,),), ([0],)),
    rejects((True,), ((0,),), ((0,),)),
    rejects((0,), ((0,),), ((0,),) * 257),
    cap_rejected,
) == (
    ViterbiResult(0, (0, 0, 0, 0)),
    ViterbiResult(31, (0, 0, 0)),
    None,
    512,
    True,
    True,
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Validation takes `O(S**2 + T*S)` time. The dense dynamic program takes
`O(T*S**2 + T*S*log(S))` time: every layer scans every transition, then sorts
at most `S` reachable prefixes to assign ranks. Backpointers use `O(T*S)`
memory, while costs and ranks use `O(S)` additional working memory.

Costs remain exact Python integers. A complete path contains one initial cost,
`T` emissions, and `T - 1` transitions, so the documented bounds cap a result
at `2*T*1_000_000`. `None` is distinct from a zero-cost choice.

This returns only one minimum-cost path. It does not enumerate ties or provide
the number of optimal paths. It also excludes probability conversion,
training, posterior inference, streaming observations, sparse transition
indexes, online updates, and `k`-best decoding. All inputs are materialized and
validated before dynamic programming starts.

## Related Snippets

<!-- catalog:related:start -->
- [Align Bounded Integer Sequences with Exact Dynamic Time Warping](align-bounded-integer-sequences-with-exact-dynamic-time-warping.md)
- [Find a Deterministic Critical Path in a Bounded Vertex-Weighted DAG](find-a-deterministic-critical-path-in-a-bounded-vertex-weighted-dag.md)
- [Find One Deterministic Shortest Path in a Bounded Non-Negatively Weighted Directed Graph](find-one-deterministic-shortest-path-in-a-bounded-non-negatively-weighted-directed-graph.md)
<!-- catalog:related:end -->
