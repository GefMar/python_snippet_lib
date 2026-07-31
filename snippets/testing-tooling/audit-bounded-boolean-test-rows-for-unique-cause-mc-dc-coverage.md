---
title: "Audit Bounded Boolean Test Rows for Unique-Cause MC/DC Coverage"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - audit-a-bounded-test-matrix-for-complete-pairwise-coverage.md
  - generate-a-deterministic-pairwise-covering-matrix-from-closed-string-parameters.md
  - ../algorithms-data-structures/build-and-evaluate-a-bounded-binary-assignment-constraint-system.md
---

# Audit Bounded Boolean Test Rows for Unique-Cause MC/DC Coverage

## Idea and Problem

Show whether observed Boolean test rows demonstrate that each condition can independently change one decision.

Unique-cause modified condition/decision coverage requires a witness pair for
every condition. The two rows in that pair must produce different decisions,
change the selected condition, and keep every other condition unchanged. That
is stronger than observing both values of each condition or covering parameter
pairs independently.

Encoding each condition tuple as a bit mask turns a witness search into a
one-bit-neighbor lookup. The audit retains original row indices for diagnostics
and resolves multiple valid witnesses with one explicit lexicographic rule.

## When to Use

Use this audit after a test harness has already recorded the atomic Boolean
condition values and final decision for each case. It is useful for checking a
small decision table, a generated truth-table subset, or exported coverage
evidence without evaluating the decision again.

Use source-aware coverage tooling when expression occurrences, short-circuit
execution, instrumentation, or traceability to program locations matters. Use
pairwise coverage for general multi-valued parameter interactions, and use a
test generator when missing witnesses must be synthesized rather than merely
reported.

## Implementation

```python
from dataclasses import dataclass

_MAX_ROW_COUNT = 4_096
_MAX_CONDITION_COUNT = 16


@dataclass(frozen=True, slots=True)
class BooleanDecisionRow:
    conditions: tuple[bool, ...]
    decision: bool


@dataclass(frozen=True, slots=True)
class UniqueCauseWitness:
    condition_index: int
    first_row_index: int
    second_row_index: int


@dataclass(frozen=True, slots=True)
class MCDCCoverageAudit:
    witnesses: tuple[UniqueCauseWitness, ...]
    missing_condition_indexes: tuple[int, ...]
    complete: bool


def audit_unique_cause_mcdc(
    rows: tuple[BooleanDecisionRow, ...],
) -> MCDCCoverageAudit:
    """Return canonical unique-cause MC/DC witnesses for observed rows."""
    if type(rows) is not tuple:
        raise TypeError("rows must be an exact tuple")
    if not 1 <= len(rows) <= _MAX_ROW_COUNT:
        raise ValueError("row count is outside the supported range")

    first_row = rows[0]
    if type(first_row) is not BooleanDecisionRow:
        raise TypeError("rows[0] must be an exact BooleanDecisionRow")
    if type(first_row.conditions) is not tuple:
        raise TypeError("rows[0].conditions must be an exact tuple")
    condition_count = len(first_row.conditions)
    if not 1 <= condition_count <= _MAX_CONDITION_COUNT:
        raise ValueError("condition count is outside the supported range")

    observed: dict[int, tuple[bool, int]] = {}
    for row_index, row in enumerate(rows):
        if type(row) is not BooleanDecisionRow:
            raise TypeError(f"rows[{row_index}] must be an exact BooleanDecisionRow")
        if type(row.conditions) is not tuple:
            raise TypeError(f"rows[{row_index}].conditions must be an exact tuple")
        if len(row.conditions) != condition_count:
            raise ValueError("every row must have the same condition count")
        if type(row.decision) is not bool:
            raise TypeError(f"rows[{row_index}].decision must be an exact boolean")

        mask = 0
        for condition_index, condition in enumerate(row.conditions):
            if type(condition) is not bool:
                raise TypeError(
                    f"rows[{row_index}].conditions[{condition_index}] must be an exact boolean"
                )
            if condition:
                mask |= 1 << condition_index

        previous = observed.get(mask)
        if previous is None:
            observed[mask] = (row.decision, row_index)
        elif previous[0] != row.decision:
            raise ValueError("one condition valuation has conflicting decisions")

    best_pairs: list[tuple[int, int] | None] = [None] * condition_count
    for mask, (decision, row_index) in observed.items():
        for condition_index in range(condition_count):
            neighbor = observed.get(mask ^ (1 << condition_index))
            if neighbor is None or neighbor[0] == decision:
                continue
            pair = tuple(sorted((row_index, neighbor[1])))
            current = best_pairs[condition_index]
            if current is None or pair < current:
                best_pairs[condition_index] = pair

    witnesses = tuple(
        UniqueCauseWitness(condition_index, pair[0], pair[1])
        for condition_index, pair in enumerate(best_pairs)
        if pair is not None
    )
    missing = tuple(
        condition_index for condition_index, pair in enumerate(best_pairs) if pair is None
    )
    return MCDCCoverageAudit(
        witnesses=witnesses,
        missing_condition_indexes=missing,
        complete=not missing,
    )
```

## Example

```python
def direct_pair_audit(
    rows: tuple[BooleanDecisionRow, ...],
) -> MCDCCoverageAudit:
    first_by_conditions: dict[tuple[bool, ...], tuple[bool, int]] = {}
    for index, row in enumerate(rows):
        previous = first_by_conditions.get(row.conditions)
        if previous is None:
            first_by_conditions[row.conditions] = (row.decision, index)
        elif previous[0] != row.decision:
            raise ValueError("one condition valuation has conflicting decisions")

    condition_count = len(rows[0].conditions)
    best: list[tuple[int, int] | None] = [None] * condition_count
    unique_rows = tuple(
        (conditions, decision, index)
        for conditions, (decision, index) in first_by_conditions.items()
    )
    for left_index, (left, left_decision, left_row) in enumerate(unique_rows):
        for right, right_decision, right_row in unique_rows[left_index + 1 :]:
            differences = tuple(
                index
                for index, (first, second) in enumerate(zip(left, right, strict=True))
                if first != second
            )
            if len(differences) != 1 or left_decision == right_decision:
                continue
            condition_index = differences[0]
            pair = tuple(sorted((left_row, right_row)))
            if best[condition_index] is None or pair < best[condition_index]:
                best[condition_index] = pair

    witnesses = tuple(
        UniqueCauseWitness(condition, pair[0], pair[1])
        for condition, pair in enumerate(best)
        if pair is not None
    )
    missing = tuple(index for index, pair in enumerate(best) if pair is None)
    return MCDCCoverageAudit(witnesses, missing, not missing)


def row_from_mask(mask: int, condition_count: int, decision: bool):
    return BooleanDecisionRow(
        tuple(bool(mask & (1 << index)) for index in range(condition_count)),
        decision,
    )


def verify_small_truth_tables() -> int:
    checked = 0
    for condition_count in range(1, 4):
        row_count = 1 << condition_count
        for function_mask in range(1 << row_count):
            rows = tuple(
                row_from_mask(
                    mask,
                    condition_count,
                    bool(function_mask & (1 << mask)),
                )
                for mask in range(row_count)
            )
            assert audit_unique_cause_mcdc(rows) == direct_pair_audit(rows)
            checked += 1

    for function_mask in range(1 << 4):
        for subset_mask in range(1, 1 << 4):
            rows = tuple(
                row_from_mask(mask, 2, bool(function_mask & (1 << mask)))
                for mask in range(4)
                if subset_mask & (1 << mask)
            )
            assert audit_unique_cause_mcdc(rows) == direct_pair_audit(rows)
            checked += 1
    return checked


and_with_duplicate = (
    row_from_mask(0, 2, False),
    row_from_mask(2, 2, False),
    row_from_mask(3, 2, True),
    row_from_mask(1, 2, False),
    row_from_mask(3, 2, True),
)
constant = tuple(row_from_mask(mask, 3, False) for mask in range(8))


def rejected(candidate: object) -> bool:
    try:
        audit_unique_cause_mcdc(candidate)
    except (TypeError, ValueError):
        return True
    return False


invalid_rows = (
    (),
    [row_from_mask(0, 1, False)],
    (BooleanDecisionRow((False,), False), BooleanDecisionRow((False,), True)),
    (BooleanDecisionRow((False,), False), BooleanDecisionRow((False, True), True)),
    (BooleanDecisionRow((0,), False),),
)

assert (
    audit_unique_cause_mcdc(and_with_duplicate),
    audit_unique_cause_mcdc(constant),
    verify_small_truth_tables(),
    sum(rejected(candidate) for candidate in invalid_rows),
) == (
    MCDCCoverageAudit(
        (
            UniqueCauseWitness(0, 1, 2),
            UniqueCauseWitness(1, 2, 3),
        ),
        (),
        True,
    ),
    MCDCCoverageAudit((), (0, 1, 2), False),
    516,
    5,
)
```

## Trade-offs and Limitations

For `R` rows and `C` conditions, validation, mask construction, and neighbor
lookups take expected `O(R * C)` time and retain `O(U + C)` state for `U`
unique valuations. The direct quadratic search in the Example is deliberately
an independent tiny-input oracle, not the production path.

Witness indices are canonical relative to declared row order. Identical rows
are allowed and only their earliest occurrence can win a witness tie. Conflicting
decisions for one condition valuation are rejected because the recorded
decision is then not a function of the supplied conditions.

This page audits unique-cause MC/DC for already observed atomic Boolean values.
It does not establish condition controllability, masking MC/DC, short-circuit
execution, repeated source-expression semantics, statement or path coverage,
instrumentation correctness, or a globally minimum covering test subset.

## Related Snippets

<!-- catalog:related:start -->
- [Audit a Bounded Test Matrix for Complete Pairwise Coverage](audit-a-bounded-test-matrix-for-complete-pairwise-coverage.md)
- [Generate a Deterministic Pairwise-Covering Matrix from Closed String Parameters](generate-a-deterministic-pairwise-covering-matrix-from-closed-string-parameters.md)
- [Build and Evaluate a Bounded Binary Assignment Constraint System](../algorithms-data-structures/build-and-evaluate-a-bounded-binary-assignment-constraint-system.md)
<!-- catalog:related:end -->
