---
title: "Audit a Bounded Transaction Schedule for Conflict Serializability"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/resolve-stable-ordering-constraints-with-topological-sort.md
  - ../algorithms-data-structures/partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md
  - commit-one-sqlite-item-mutation-and-its-checkpoint-atomically.md
---

# Audit a Bounded Transaction Schedule for Conflict Serializability

## Idea and Problem

Derive a transaction precedence graph from a bounded read/write schedule, then either produce a deterministic equivalent serial order or expose one concrete cycle.

Two operations conflict when they belong to different transactions, access the
same item, and at least one is a write. Their schedule order creates a directed
precedence edge. An acyclic graph proves conflict serializability; a cycle proves
that no serial order can preserve every conflict. If several operation pairs
support one edge, the pair minimizing `(after_index, before_index)` is its
canonical earliest evidence.

## When to Use

Use this audit for small, already captured schedules in teaching tools, model
tests, or deterministic diagnostics. It is particularly useful when a Boolean
answer is insufficient: every edge retains its earliest supporting operation
pair, and a negative result includes a cycle that can be explained.

Use a database's concurrency diagnostics for live systems. This model covers
only single-version reads and writes; it does not model commits, aborts, locks,
MVCC visibility, predicate reads, recoverability, or view serializability.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import StrEnum
from heapq import heappop, heappush

_MAX_TRANSACTIONS = 64
_MAX_OPERATIONS = 4_096
_MAX_ITEMS = 512
_IDENTIFIER = re.compile(r"[A-Za-z0-9](?:[A-Za-z0-9._-]{0,63})", re.ASCII).fullmatch


class Access(StrEnum):
    READ = "read"
    WRITE = "write"


@dataclass(frozen=True, slots=True)
class ScheduleOperation:
    transaction: str
    item: str
    access: Access


@dataclass(frozen=True, slots=True)
class ConflictEvidence:
    before_index: int
    after_index: int
    from_transaction: str
    to_transaction: str
    item: str
    before_access: Access
    after_access: Access


@dataclass(frozen=True, slots=True)
class SerializabilityAudit:
    conflicts: tuple[ConflictEvidence, ...]
    serial_order: tuple[str, ...] | None
    cycle: tuple[str, ...] | None


def _validate_identifier(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if _IDENTIFIER(value) is None:
        raise ValueError(f"{field} must be a conservative ASCII identifier of 1..64 bytes")
    return value


def audit_conflict_serializability(
    transactions: tuple[str, ...],
    schedule: tuple[ScheduleOperation, ...],
) -> SerializabilityAudit:
    if type(transactions) is not tuple:
        raise TypeError("transactions must be an exact tuple")
    if not 1 <= len(transactions) <= _MAX_TRANSACTIONS:
        raise ValueError("transaction count is outside 1..64")
    checked_transactions = tuple(
        _validate_identifier(value, field=f"transactions[{index}]")
        for index, value in enumerate(transactions)
    )
    if len(set(checked_transactions)) != len(checked_transactions):
        raise ValueError("transaction identifiers must be unique")

    if type(schedule) is not tuple:
        raise TypeError("schedule must be an exact tuple")
    if len(schedule) > _MAX_OPERATIONS:
        raise ValueError("operation count exceeds 4096")

    transaction_index = {
        transaction: index
        for index, transaction in enumerate(checked_transactions)
    }
    checked_operations: list[ScheduleOperation] = []
    for index, operation in enumerate(schedule):
        if type(operation) is not ScheduleOperation:
            raise TypeError(f"schedule[{index}] must be an exact ScheduleOperation")
        transaction = _validate_identifier(
            operation.transaction,
            field=f"schedule[{index}].transaction",
        )
        if transaction not in transaction_index:
            raise ValueError(f"schedule[{index}] names an undeclared transaction")
        item = _validate_identifier(operation.item, field=f"schedule[{index}].item")
        if type(operation.access) is not Access:
            raise TypeError(f"schedule[{index}].access must be an exact Access")
        checked_operations.append(ScheduleOperation(transaction, item, operation.access))
    if len({operation.item for operation in checked_operations}) > _MAX_ITEMS:
        raise ValueError("schedule contains more than 512 distinct items")

    witnesses: dict[tuple[int, int], ConflictEvidence] = {}
    for after_index, after in enumerate(checked_operations):
        for before_index in range(after_index):
            before = checked_operations[before_index]
            if (
                before.transaction == after.transaction
                or before.item != after.item
                or (before.access is Access.READ and after.access is Access.READ)
            ):
                continue
            source = transaction_index[before.transaction]
            target = transaction_index[after.transaction]
            witnesses.setdefault(
                (source, target),
                ConflictEvidence(
                    before_index,
                    after_index,
                    before.transaction,
                    after.transaction,
                    before.item,
                    before.access,
                    after.access,
                ),
            )

    transaction_count = len(checked_transactions)
    neighbors = [set() for _ in range(transaction_count)]
    indegree = [0] * transaction_count
    for source, target in witnesses:
        neighbors[source].add(target)
        indegree[target] += 1

    available = [index for index, degree in enumerate(indegree) if degree == 0]
    serial_indexes: list[int] = []
    while available:
        source = heappop(available)
        serial_indexes.append(source)
        for target in sorted(neighbors[source]):
            indegree[target] -= 1
            if indegree[target] == 0:
                heappush(available, target)

    conflicts = tuple(witnesses[edge] for edge in sorted(witnesses))
    if len(serial_indexes) == transaction_count:
        return SerializabilityAudit(
            conflicts,
            tuple(checked_transactions[index] for index in serial_indexes),
            None,
        )

    color = [0] * transaction_count
    parent = [-1] * transaction_count
    cycle_indexes: tuple[int, ...] | None = None

    def visit(source: int) -> bool:
        nonlocal cycle_indexes
        color[source] = 1
        for target in sorted(neighbors[source]):
            if color[target] == 0:
                parent[target] = source
                if visit(target):
                    return True
            elif color[target] == 1:
                trail = [source]
                while trail[-1] != target:
                    trail.append(parent[trail[-1]])
                trail.reverse()
                trail.append(target)
                cycle_indexes = tuple(trail)
                return True
        color[source] = 2
        return False

    for root in range(transaction_count):
        if color[root] == 0 and visit(root):
            break

    assert cycle_indexes is not None
    return SerializabilityAudit(
        conflicts,
        None,
        tuple(checked_transactions[index] for index in cycle_indexes),
    )
```

## Example

```python
def independent_edges(
    schedule: tuple[ScheduleOperation, ...],
) -> set[tuple[str, str]]:
    return {
        (before.transaction, after.transaction)
        for left, before in enumerate(schedule)
        for after in schedule[left + 1 :]
        if before.transaction != after.transaction
        and before.item == after.item
        and Access.WRITE in (before.access, after.access)
    }


def valid_serial_orders(
    transactions: tuple[str, ...],
    edges: set[tuple[str, str]],
) -> tuple[tuple[str, ...], ...]:
    from itertools import permutations

    result = []
    for order in permutations(transactions):
        position = {transaction: index for index, transaction in enumerate(order)}
        if all(position[source] < position[target] for source, target in edges):
            result.append(order)
    return tuple(result)


transactions = ("T1", "T2", "T3", "T4")
serializable = (
    ScheduleOperation("T2", "item-a", Access.WRITE),
    ScheduleOperation("T1", "item-b", Access.READ),
    ScheduleOperation("T3", "item-a", Access.READ),
    ScheduleOperation("T1", "item-c", Access.WRITE),
    ScheduleOperation("T4", "item-c", Access.READ),
)
edges = independent_edges(serializable)
audit = audit_conflict_serializability(transactions, serializable)
valid_orders = valid_serial_orders(transactions, edges)

cyclic = (
    ScheduleOperation("T1", "x", Access.WRITE),
    ScheduleOperation("T2", "x", Access.READ),
    ScheduleOperation("T2", "y", Access.WRITE),
    ScheduleOperation("T1", "y", Access.READ),
)
cyclic_audit = audit_conflict_serializability(("T1", "T2"), cyclic)
cyclic_edges = independent_edges(cyclic)
assert cyclic_audit.cycle is not None
cycle_arcs = set(
    zip(cyclic_audit.cycle[:-1], cyclic_audit.cycle[1:], strict=True)
)
empty_audit = audit_conflict_serializability(("T1", "T2"), ())
read_only_audit = audit_conflict_serializability(
    ("T1", "T2"),
    (
        ScheduleOperation("T1", "x", Access.READ),
        ScheduleOperation("T2", "x", Access.READ),
    ),
)
repeated = audit_conflict_serializability(
    ("T1", "T2"),
    (
        ScheduleOperation("T1", "x", Access.WRITE),
        ScheduleOperation("T1", "y", Access.WRITE),
        ScheduleOperation("T2", "y", Access.READ),
        ScheduleOperation("T2", "x", Access.READ),
    ),
)

assert (
    edges == {("T2", "T3"), ("T1", "T4")}
    and audit.serial_order == min(
        valid_orders,
        key=lambda order: tuple(transactions.index(value) for value in order),
    )
    and audit.cycle is None
    and {(item.from_transaction, item.to_transaction) for item in audit.conflicts}
    == edges
    and cyclic_audit.serial_order is None
    and cycle_arcs <= cyclic_edges
    and all(
        (item.from_transaction, item.to_transaction) in cyclic_edges
        and item.before_index < item.after_index
        and cyclic[item.before_index].transaction == item.from_transaction
        and cyclic[item.after_index].transaction == item.to_transaction
        and cyclic[item.before_index].item == cyclic[item.after_index].item == item.item
        and cyclic[item.before_index].access is item.before_access
        and cyclic[item.after_index].access is item.after_access
        for item in cyclic_audit.conflicts
    )
    and empty_audit.serial_order == ("T1", "T2")
    and empty_audit.conflicts == ()
    and read_only_audit.conflicts == ()
    and repeated.conflicts[0].before_index == 1
    and repeated.conflicts[0].after_index == 2
)
```

## Trade-offs and Limitations

For `M` operations, pairwise conflict discovery takes `O(M**2)` time. The
precedence graph then uses `O(T**2)` possible edges for `T` transactions; both
dimensions are explicitly capped. Keeping only the earliest witness per edge
bounds the diagnostic result without weakening the graph; "earliest" compares
the later schedule index first and the earlier index second.

The topological order is the smallest available declared transaction at every
step, and DFS roots and neighbors use that same declared order. These rules make
the result reproducible, but another valid serial order or cycle can exist.
Identifiers are labels only; the audit knows nothing about the values read or
written, isolation levels, or effects outside the supplied schedule.

## Related Snippets

<!-- catalog:related:start -->
- [Resolve Stable Ordering Constraints with Topological Sort](../algorithms-data-structures/resolve-stable-ordering-constraints-with-topological-sort.md)
- [Partition a Bounded Directed Graph into Deterministic Strongly Connected Components](../algorithms-data-structures/partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md)
- [Commit One SQLite Item Mutation and Its Checkpoint Atomically](commit-one-sqlite-item-mutation-and-its-checkpoint-atomically.md)
<!-- catalog:related:end -->
