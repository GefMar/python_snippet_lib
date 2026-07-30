---
title: "Plan Collision-Free Renames in a Bounded Flat File Namespace"
snippet_type: algorithm
use_cases:
  - automation
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-and-apply-a-deterministic-mapping-delta.md
  - publish-a-new-posix-file-without-replacement-and-sync-directory-entries.md
  - ../testing-tooling/build-a-collision-audited-artifact-copy-plan.md
---

# Plan Collision-Free Renames in a Bounded Flat File Namespace

## Idea and Problem

Order an injective rename plan so every destination is free at the moment it is used, including chains and cycles.

The caller supplies a complete logical occupancy snapshot, an exact
source-to-target mapping, and one known-free temporary name. Renames whose
targets are already free run first and progressively unblock their
predecessors. When only cycles remain, one deterministic move to the temporary
name turns a cycle into a chain.

## When to Use

Use this planner after discovering and validating a small flat namespace but
before a separate no-replacement executor performs moves. It is useful for
reviewing migrations, fixture renames, and identity-preserving rotations where
overwriting an occupied target would lose data.

Treat the result as a plan over one frozen snapshot, not a filesystem safety
capability. A real executor must revalidate occupancy, containment, object
identity, permissions, and failure recovery immediately before and during
execution. Use platform-specific primitives when moves must be atomic or
resistant to concurrent namespace changes.

## Implementation

```python
from dataclasses import dataclass

_MAX_OCCUPIED_NAMES = 256
_MAX_RENAMES = 128
_MAX_NAME_BYTES = 256
_MAX_AGGREGATE_NAME_BYTES = 65_536


@dataclass(frozen=True, slots=True)
class RenameIntent:
    source: str
    target: str


@dataclass(frozen=True, slots=True)
class RenameStep:
    source: str
    target: str


def _validated_name(value: object, *, field: str) -> tuple[str, int]:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    try:
        size = len(value.encode("utf-8"))
    except UnicodeEncodeError:
        raise ValueError(f"{field} must be UTF-8 encodable") from None
    if not 1 <= size <= _MAX_NAME_BYTES:
        raise ValueError(f"{field} is outside its UTF-8 byte limit")
    return value, size


def plan_collision_free_renames(
    occupied_names: tuple[str, ...],
    renames: tuple[RenameIntent, ...],
    temporary_name: str,
) -> tuple[RenameStep, ...]:
    if type(occupied_names) is not tuple:
        raise TypeError("occupied_names must be an exact tuple")
    if not 1 <= len(occupied_names) <= _MAX_OCCUPIED_NAMES:
        raise ValueError("occupied name count is outside 1..256")

    checked_occupied: list[str] = []
    aggregate_name_bytes = 0
    for index, name in enumerate(occupied_names):
        checked, size = _validated_name(name, field=f"occupied_names[{index}]")
        checked_occupied.append(checked)
        aggregate_name_bytes += size
    if len(set(checked_occupied)) != len(checked_occupied):
        raise ValueError("occupied_names must be unique")

    if type(renames) is not tuple:
        raise TypeError("renames must be an exact tuple")
    if not 1 <= len(renames) <= _MAX_RENAMES:
        raise ValueError("rename count is outside 1..128")

    checked_renames: list[RenameIntent] = []
    sources: set[str] = set()
    targets: set[str] = set()
    for index, rename in enumerate(renames):
        field = f"renames[{index}]"
        if type(rename) is not RenameIntent:
            raise TypeError(f"{field} must be an exact RenameIntent")
        source, source_size = _validated_name(
            rename.source,
            field=f"{field}.source",
        )
        target, target_size = _validated_name(
            rename.target,
            field=f"{field}.target",
        )
        aggregate_name_bytes += source_size + target_size
        if source == target:
            raise ValueError(f"{field} must change the logical name")
        if source in sources:
            raise ValueError("rename sources must be unique")
        if target in targets:
            raise ValueError("rename targets must be unique")
        sources.add(source)
        targets.add(target)
        checked_renames.append(RenameIntent(source, target))

    checked_temporary, temporary_size = _validated_name(
        temporary_name,
        field="temporary_name",
    )
    aggregate_name_bytes += temporary_size
    if aggregate_name_bytes > _MAX_AGGREGATE_NAME_BYTES:
        raise ValueError("input names exceed the aggregate UTF-8 byte limit")

    occupied = set(checked_occupied)
    if not sources <= occupied:
        raise ValueError("every rename source must be occupied")
    if any(target in occupied and target not in sources for target in targets):
        raise ValueError("an occupied target must also be a rename source")
    if (
        checked_temporary in occupied
        or checked_temporary in sources
        or checked_temporary in targets
    ):
        raise ValueError("temporary_name must be free and reserved for the plan")

    pending = {
        rename.source: rename.target
        for rename in checked_renames
    }
    steps: list[RenameStep] = []
    while pending:
        ready_sources = sorted(
            source
            for source, target in pending.items()
            if target not in occupied
        )
        if ready_sources:
            source = ready_sources[0]
            target = pending.pop(source)
        else:
            source = min(pending)
            target = checked_temporary
            original_target = pending.pop(source)
            pending[checked_temporary] = original_target

        occupied.remove(source)
        occupied.add(target)
        steps.append(RenameStep(source, target))

    return tuple(steps)
```

## Example

```python
def simulate_plan(
    occupied_names: tuple[str, ...],
    renames: tuple[RenameIntent, ...],
    steps: tuple[RenameStep, ...],
) -> dict[str, str]:
    original = {name: f"identity:{name}" for name in occupied_names}
    current = dict(original)
    for step in steps:
        assert step.source in current
        assert step.target not in current
        current[step.target] = current.pop(step.source)

    expected = {
        name: identity
        for name, identity in original.items()
        if name not in {rename.source for rename in renames}
    }
    expected.update(
        (rename.target, original[rename.source])
        for rename in renames
    )
    assert current == expected
    return current


def check_small_injective_plans() -> int:
    from itertools import combinations, permutations

    occupied = ("a", "b", "c")
    candidate_targets = (*occupied, "d")
    checked = 0
    for source_count in range(1, len(occupied) + 1):
        for selected_sources in combinations(occupied, source_count):
            for selected_targets in permutations(candidate_targets, source_count):
                renames = tuple(
                    RenameIntent(source, target)
                    for source, target in zip(
                        selected_sources,
                        selected_targets,
                        strict=True,
                    )
                )
                if any(rename.source == rename.target for rename in renames):
                    continue
                if any(
                    rename.target in occupied
                    and rename.target not in selected_sources
                    for rename in renames
                ):
                    continue
                steps = plan_collision_free_renames(occupied, renames, "temp")
                simulate_plan(occupied, renames, steps)
                assert steps == plan_collision_free_renames(
                    occupied,
                    tuple(reversed(renames)),
                    "temp",
                )
                checked += 1
    return checked


sample_renames = (
    RenameIntent("a", "b"),
    RenameIntent("b", "a"),
    RenameIntent("c", "d"),
)
sample_steps = plan_collision_free_renames(
    ("a", "b", "c", "external"),
    sample_renames,
    "~temporary",
)
sample_state = simulate_plan(
    ("a", "b", "c", "external"),
    sample_renames,
    sample_steps,
)

disjoint_cycles = (
    RenameIntent("a", "b"),
    RenameIntent("b", "a"),
    RenameIntent("c", "d"),
    RenameIntent("d", "c"),
)
disjoint_steps = plan_collision_free_renames(
    ("a", "b", "c", "d"),
    disjoint_cycles,
    "temporary",
)
disjoint_state = simulate_plan(
    ("a", "b", "c", "d"),
    disjoint_cycles,
    disjoint_steps,
)

maximum_names = tuple(
    f"{index:03d}" + "x" * 253
    for index in range(_MAX_OCCUPIED_NAMES)
)
try:
    plan_collision_free_renames(
        maximum_names,
        (RenameIntent(maximum_names[0], "free"),),
        "temporary",
    )
except ValueError:
    aggregate_cap_enforced = True
else:
    aggregate_cap_enforced = False

assert (
    check_small_injective_plans() == 23
    and sample_steps
    == (
        RenameStep("c", "d"),
        RenameStep("a", "~temporary"),
        RenameStep("b", "a"),
        RenameStep("~temporary", "b"),
    )
    and sample_state["a"] == "identity:b"
    and sample_state["b"] == "identity:a"
    and sum(step.target == "temporary" for step in disjoint_steps) == 2
    and "temporary" not in disjoint_state
    and aggregate_cap_enforced
)
```

## Trade-offs and Limitations

With at most `R = 128` renames, repeatedly scanning and sorting the pending map
takes `O(R^2 log R)` time; occupancy, pending state, and output use `O(O + R)`
memory for `O` occupied names. Acyclic plans emit `R` steps, and each disjoint
cycle adds one temporary move.

Python's Unicode string order is the deterministic tie rule. It is not a
filesystem collation rule, and names are deliberately opaque rather than path
objects. A failure during real execution can leave a partially moved namespace;
the planner provides neither rollback nor durable journaling and cannot detect
an external change after its snapshot.

## Related Snippets

<!-- catalog:related:start -->
- [Build and Apply a Deterministic Mapping Delta](build-and-apply-a-deterministic-mapping-delta.md)
- [Publish a New POSIX File Without Replacement and Sync Directory Entries](publish-a-new-posix-file-without-replacement-and-sync-directory-entries.md)
- [Build a Collision-Audited Artifact Copy Plan](../testing-tooling/build-a-collision-audited-artifact-copy-plan.md)
<!-- catalog:related:end -->
