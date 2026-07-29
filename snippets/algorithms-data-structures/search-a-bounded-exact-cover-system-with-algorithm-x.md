---
title: "Search a Bounded Exact-Cover System with Algorithm X"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-and-evaluate-a-bounded-binary-assignment-constraint-system.md
  - find-one-deterministic-satisfying-assignment-for-bounded-2-cnf.md
  - find-a-deterministic-maximum-cardinality-matching-in-a-bounded-bipartite-graph.md
---

# Search a Bounded Exact-Cover System with Algorithm X

## Idea and Problem

Search for disjoint rows that cover every declared column exactly once without confusing a search limit with proof that no solution exists.

Algorithm X performs depth-first backtracking. At each state it chooses an
uncovered column with the fewest compatible rows, then tries those rows in a
fixed order. Integer masks make overlap and completion checks explicit.

Canonical name sorting makes the search independent of input order. A
three-state result distinguishes a found cover, a completely explored
unsatisfiable system, and a search stopped before another state could be
entered.

## When to Use

Use this bounded solver for small assignment, selection, tiling, or test-oracle
problems that naturally mean “choose rows so every constraint appears exactly
once.” Give every logical row and column a stable, descriptive name.

The combined incidence and search-state cap is important because exact cover
is NP-complete. Use a specialized solver when instances are large, need
secondary constraints, or must enumerate or optimize solutions.

## Implementation

```python
from dataclasses import dataclass
from enum import StrEnum

_MAX_COLUMNS = 32
_MAX_ROWS = 256
_MAX_INCIDENCES = 4_096
_MAX_SEARCH_STATES = 100_000
_MAX_SCAN_WORK = 5_000_000
_MAX_NAME_LENGTH = 64
_NAME_CHARACTERS = frozenset(
    "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz._-"
)


class ExactCoverStatus(StrEnum):
    FOUND = "found"
    UNSATISFIABLE = "unsatisfiable"
    BUDGET_EXHAUSTED = "budget_exhausted"


@dataclass(frozen=True, slots=True)
class ExactCoverRow:
    name: str
    columns: tuple[str, ...]


@dataclass(frozen=True, slots=True)
class ExactCoverResult:
    status: ExactCoverStatus
    selected_rows: tuple[str, ...]
    visited_search_states: int


def _validate_name(kind: str, value: object) -> str:
    if type(value) is not str:
        raise TypeError(f"{kind} name must be an exact string")
    if not 1 <= len(value) <= _MAX_NAME_LENGTH:
        raise ValueError(f"{kind} name length is outside the supported range")
    if any(character not in _NAME_CHARACTERS for character in value):
        raise ValueError(f"{kind} name contains a forbidden character")
    return value


def search_exact_cover(
    columns: tuple[str, ...],
    rows: tuple[ExactCoverRow, ...],
    *,
    max_search_states: int,
) -> ExactCoverResult:
    """Return one deterministic exact cover or an explicit non-found status."""
    if type(columns) is not tuple:
        raise TypeError("columns must be an exact tuple")
    if not 1 <= len(columns) <= _MAX_COLUMNS:
        raise ValueError("column count is outside the supported range")
    if type(rows) is not tuple:
        raise TypeError("rows must be an exact tuple")
    if len(rows) > _MAX_ROWS:
        raise ValueError("row count exceeds the supported limit")
    if type(max_search_states) is not int:
        raise TypeError("max_search_states must be an exact integer")
    if not 1 <= max_search_states <= _MAX_SEARCH_STATES:
        raise ValueError("search-state budget is outside the supported range")

    validated_columns = tuple(
        _validate_name("column", column)
        for column in columns
    )
    if len(set(validated_columns)) != len(validated_columns):
        raise ValueError("column names must be unique")
    declared_columns = frozenset(validated_columns)

    validated_rows: list[ExactCoverRow] = []
    row_names: set[str] = set()
    incidence_count = 0
    for row in rows:
        if type(row) is not ExactCoverRow:
            raise TypeError("rows must contain exact ExactCoverRow values")
        row_name = _validate_name("row", row.name)
        if row_name in row_names:
            raise ValueError("row names must be unique")
        row_names.add(row_name)
        if type(row.columns) is not tuple:
            raise TypeError("row columns must be an exact tuple")
        if not 1 <= len(row.columns) <= _MAX_COLUMNS:
            raise ValueError("row column count is outside the supported range")
        row_columns = tuple(
            _validate_name("row column", column)
            for column in row.columns
        )
        if len(set(row_columns)) != len(row_columns):
            raise ValueError("columns within a row must be unique")
        if any(column not in declared_columns for column in row_columns):
            raise ValueError("row refers to an undeclared column")
        incidence_count += len(row_columns)
        if incidence_count > _MAX_INCIDENCES:
            raise ValueError("incidence count exceeds the supported limit")
        validated_rows.append(ExactCoverRow(row_name, row_columns))

    scan_width = len(validated_columns) + incidence_count
    if max_search_states * scan_width > _MAX_SCAN_WORK:
        raise ValueError("combined search scan budget exceeds the supported limit")

    canonical_columns = tuple(sorted(validated_columns))
    column_indexes = {
        column: index
        for index, column in enumerate(canonical_columns)
    }
    canonical_rows = tuple(sorted(validated_rows, key=lambda row: row.name))
    row_masks = tuple(
        sum(1 << column_indexes[column] for column in row.columns)
        for row in canonical_rows
    )
    rows_by_column: list[list[int]] = [
        []
        for _ in canonical_columns
    ]
    for row_index, row_mask in enumerate(row_masks):
        for column_index in range(len(canonical_columns)):
            if row_mask & (1 << column_index):
                rows_by_column[column_index].append(row_index)

    complete_mask = (1 << len(canonical_columns)) - 1
    visited_search_states = 0
    budget_exhausted = False

    def visit(
        covered_mask: int,
        selected_indexes: tuple[int, ...],
    ) -> tuple[int, ...] | None:
        nonlocal budget_exhausted, visited_search_states

        if visited_search_states >= max_search_states:
            budget_exhausted = True
            return None
        visited_search_states += 1

        if covered_mask == complete_mask:
            return selected_indexes

        best_candidates: tuple[int, ...] | None = None
        for column_index, incident_rows in enumerate(rows_by_column):
            if covered_mask & (1 << column_index):
                continue
            compatible = tuple(
                row_index
                for row_index in incident_rows
                if not row_masks[row_index] & covered_mask
            )
            if best_candidates is None or len(compatible) < len(best_candidates):
                best_candidates = compatible
            if not best_candidates:
                break

        if not best_candidates:
            return None

        for row_index in best_candidates:
            solution = visit(
                covered_mask | row_masks[row_index],
                (*selected_indexes, row_index),
            )
            if solution is not None:
                return solution
            if budget_exhausted:
                return None
        return None

    solution = visit(0, ())
    if solution is not None:
        return ExactCoverResult(
            ExactCoverStatus.FOUND,
            tuple(sorted(canonical_rows[index].name for index in solution)),
            visited_search_states,
        )
    if budget_exhausted:
        return ExactCoverResult(
            ExactCoverStatus.BUDGET_EXHAUSTED,
            (),
            visited_search_states,
        )
    return ExactCoverResult(
        ExactCoverStatus.UNSATISFIABLE,
        (),
        visited_search_states,
    )
```

## Example

```python
columns = ("D", "B", "A", "C")
rows = (
    ExactCoverRow("row-4", ("D", "B")),
    ExactCoverRow("row-3", ("C", "A")),
    ExactCoverRow("row-2", ("D", "C")),
    ExactCoverRow("row-1", ("B", "A")),
)

result = search_exact_cover(
    columns,
    rows,
    max_search_states=100,
)
assert result.status is ExactCoverStatus.FOUND
assert result.selected_rows == ("row-1", "row-2")

same_system_reordered = search_exact_cover(
    tuple(reversed(columns)),
    tuple(reversed(rows)),
    max_search_states=100,
)
assert same_system_reordered == result

unsatisfiable = search_exact_cover(
    ("A", "B"),
    (ExactCoverRow("only-a", ("A",)),),
    max_search_states=10,
)
assert unsatisfiable.status is ExactCoverStatus.UNSATISFIABLE

exhausted = search_exact_cover(
    ("A",),
    (ExactCoverRow("covers-a", ("A",)),),
    max_search_states=1,
)
assert exhausted.status is ExactCoverStatus.BUDGET_EXHAUSTED
```

## Trade-offs and Limitations

Canonicalization sorts names before search. Each admitted state scans at most
all columns and incidences, so search performs
`O(B * (C + I))` simple work for budget `B`, column count `C`, and incidence
count `I`. The combined preflight cap limits that product to five million.
Backtracking state depth is at most the number of columns.

The first solution follows the declared minimum-compatible-column and lexical
row rules. It is deterministic but is not promised to use the fewest rows.
`BUDGET_EXHAUSTED` means the instance remains unresolved; only a complete
bounded traversal can return `UNSATISFIABLE`.

This implementation does not use the dancing-links optimization. It provides
no secondary columns, row weights, minimum-cardinality objective, all-solution
enumeration, resumable search, parallelism, or proof certificate.

## Related Snippets

<!-- catalog:related:start -->
- [Build and Evaluate a Bounded Binary Assignment Constraint System](build-and-evaluate-a-bounded-binary-assignment-constraint-system.md)
- [Find One Deterministic Satisfying Assignment for Bounded 2-CNF](find-one-deterministic-satisfying-assignment-for-bounded-2-cnf.md)
- [Find a Deterministic Maximum-Cardinality Matching in a Bounded Bipartite Graph](find-a-deterministic-maximum-cardinality-matching-in-a-bounded-bipartite-graph.md)
<!-- catalog:related:end -->
