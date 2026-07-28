---
title: "Plan Verified Staged Partition Publication"
snippet_type: algorithm
use_cases:
  - automation
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - validate-a-bounded-chunk-manifest-before-a-conditional-version-switch.md
  - validate-a-bounded-stage-verify-pointer-switch-log.md
  - compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md
---

# Plan Verified Staged Partition Publication

## Idea and Problem

Classify every desired partition from complete, bounded observations without hiding safe progress behind unrelated conflicts.

Each result has exactly one action and one reason. A matching target is already
current unless matching staging remains to be cleaned up. A differing target is
only a promotion candidate when staging exactly matches the desired content and
the target revision and token equal the caller's expected observation. Missing,
mismatched, or contradictory evidence blocks only that partition.

## When to Use

Use this algorithm after another layer has produced immutable desired records
and optional staging and target observations. Partition and schema IDs are
exact conservative ASCII identifiers; content identity includes schema,
lowercase SHA-256 digest, row count, and byte count. All tuples must be in
strict partition-ID order so the returned order is reproducible.

This is a snapshot classifier, not publication authority. An executor must
conditionally revalidate every referenced observation immediately before an
action and then reobserve the partition after every attempt. A timeout or lost
response is ambiguous: it must not be interpreted as proof that promotion or
cleanup failed.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import StrEnum

_MAX_PARTITIONS = 256
_MAX_COUNT = 2**63 - 1
_PARTITION_ID = re.compile(r"[a-z0-9][a-z0-9._-]{0,63}", re.ASCII).fullmatch
_SCHEMA_ID = re.compile(r"[A-Za-z0-9][A-Za-z0-9._:-]{0,63}", re.ASCII).fullmatch
_TOKEN = re.compile(r"[A-Za-z0-9][A-Za-z0-9._:-]{0,95}", re.ASCII).fullmatch
_DIGEST = re.compile(r"[0-9a-f]{64}", re.ASCII).fullmatch


@dataclass(frozen=True, slots=True)
class PartitionContent:
    schema_id: str
    content_digest: str
    row_count: int
    byte_count: int


@dataclass(frozen=True, slots=True)
class TargetStamp:
    revision: int
    token: str


@dataclass(frozen=True, slots=True)
class DesiredPartition:
    partition_id: str
    content: PartitionContent
    expected_target: TargetStamp


@dataclass(frozen=True, slots=True)
class StagingObservation:
    partition_id: str
    content: PartitionContent


@dataclass(frozen=True, slots=True)
class TargetObservation:
    partition_id: str
    content: PartitionContent
    stamp: TargetStamp


class PublicationAction(StrEnum):
    PROMOTE_CANDIDATE = "PROMOTE_CANDIDATE"
    ALREADY_CURRENT = "ALREADY_CURRENT"
    CLEANUP_CANDIDATE = "CLEANUP_CANDIDATE"
    BLOCKED = "BLOCKED"


@dataclass(frozen=True, slots=True)
class PartitionDecision:
    partition_id: str
    action: PublicationAction
    reason: str
    observed_target: TargetStamp | None


def _checked_text(value: object, *, field: str, pattern: object) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if pattern(value) is None:  # type: ignore[operator]
        raise ValueError(f"{field} has invalid syntax or length")
    return value


def _checked_count(value: object, *, field: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer")
    if not 0 <= value <= _MAX_COUNT:
        raise ValueError(f"{field} is outside the supported range")
    return value


def _checked_content(value: object, *, field: str) -> PartitionContent:
    if type(value) is not PartitionContent:
        raise TypeError(f"{field} must be an exact PartitionContent")
    return PartitionContent(
        schema_id=_checked_text(
            value.schema_id,
            field=f"{field}.schema_id",
            pattern=_SCHEMA_ID,
        ),
        content_digest=_checked_text(
            value.content_digest,
            field=f"{field}.content_digest",
            pattern=_DIGEST,
        ),
        row_count=_checked_count(value.row_count, field=f"{field}.row_count"),
        byte_count=_checked_count(value.byte_count, field=f"{field}.byte_count"),
    )


def _checked_stamp(value: object, *, field: str) -> TargetStamp:
    if type(value) is not TargetStamp:
        raise TypeError(f"{field} must be an exact TargetStamp")
    return TargetStamp(
        revision=_checked_count(value.revision, field=f"{field}.revision"),
        token=_checked_text(value.token, field=f"{field}.token", pattern=_TOKEN),
    )


def _check_order(ids: tuple[str, ...], *, field: str) -> None:
    if len(set(ids)) != len(ids):
        raise ValueError(f"{field} partition IDs must be unique")
    if ids != tuple(sorted(ids)):
        raise ValueError(f"{field} must be in strict partition-ID order")


def _checked_desired(value: object) -> tuple[DesiredPartition, ...]:
    if type(value) is not tuple:
        raise TypeError("desired must be an exact tuple")
    if not 1 <= len(value) <= _MAX_PARTITIONS:
        raise ValueError(f"desired must contain 1 to {_MAX_PARTITIONS} records")
    checked: list[DesiredPartition] = []
    for index, item in enumerate(value):
        field = f"desired[{index}]"
        if type(item) is not DesiredPartition:
            raise TypeError(f"{field} must be an exact DesiredPartition")
        checked.append(
            DesiredPartition(
                _checked_text(
                    item.partition_id,
                    field=f"{field}.partition_id",
                    pattern=_PARTITION_ID,
                ),
                _checked_content(item.content, field=f"{field}.content"),
                _checked_stamp(item.expected_target, field=f"{field}.expected_target"),
            )
        )
    frozen = tuple(checked)
    _check_order(tuple(item.partition_id for item in frozen), field="desired")
    return frozen


def _checked_observations(
    value: object,
    *,
    record_type: type[StagingObservation] | type[TargetObservation],
    desired_ids: set[str],
    field: str,
) -> tuple[StagingObservation | TargetObservation, ...]:
    if type(value) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if len(value) > _MAX_PARTITIONS:
        raise ValueError(f"{field} contains too many records")
    checked: list[StagingObservation | TargetObservation] = []
    for index, item in enumerate(value):
        item_field = f"{field}[{index}]"
        if type(item) is not record_type:
            raise TypeError(f"{item_field} has the wrong exact record type")
        partition_id = _checked_text(
            item.partition_id,
            field=f"{item_field}.partition_id",
            pattern=_PARTITION_ID,
        )
        content = _checked_content(item.content, field=f"{item_field}.content")
        if record_type is StagingObservation:
            checked.append(StagingObservation(partition_id, content))
        else:
            checked.append(
                TargetObservation(
                    partition_id,
                    content,
                    _checked_stamp(item.stamp, field=f"{item_field}.stamp"),
                )
            )
    frozen = tuple(checked)
    ids = tuple(item.partition_id for item in frozen)
    _check_order(ids, field=field)
    if not set(ids) <= desired_ids:
        raise ValueError(f"{field} contains an undeclared partition ID")
    return frozen


def _decision(
    desired: DesiredPartition,
    staging: StagingObservation | None,
    target: TargetObservation | None,
) -> PartitionDecision:
    observed = None if target is None else target.stamp
    if target is None:
        return PartitionDecision(
            desired.partition_id,
            PublicationAction.BLOCKED,
            "target observation is missing",
            observed,
        )
    if target.content == desired.content:
        if staging is None:
            return PartitionDecision(
                desired.partition_id,
                PublicationAction.ALREADY_CURRENT,
                "target exactly matches desired content and staging is absent",
                observed,
            )
        if staging.content == desired.content:
            return PartitionDecision(
                desired.partition_id,
                PublicationAction.CLEANUP_CANDIDATE,
                "target and remaining staging both exactly match desired content",
                observed,
            )
        return PartitionDecision(
            desired.partition_id,
            PublicationAction.BLOCKED,
            "remaining staging does not exactly match desired content",
            observed,
        )
    if target.content.content_digest == desired.content.content_digest:
        return PartitionDecision(
            desired.partition_id,
            PublicationAction.BLOCKED,
            "target digest agrees but schema or counts contradict desired content",
            observed,
        )
    if staging is None:
        return PartitionDecision(
            desired.partition_id,
            PublicationAction.BLOCKED,
            "target differs and staging observation is missing",
            observed,
        )
    if staging.content != desired.content:
        return PartitionDecision(
            desired.partition_id,
            PublicationAction.BLOCKED,
            "staging does not exactly match desired content",
            observed,
        )
    if target.stamp != desired.expected_target:
        return PartitionDecision(
            desired.partition_id,
            PublicationAction.BLOCKED,
            "target revision or token differs from the expected observation",
            observed,
        )
    return PartitionDecision(
        desired.partition_id,
        PublicationAction.PROMOTE_CANDIDATE,
        "matching staging is promotable under the expected target observation",
        observed,
    )


def plan_partition_publication(
    desired: tuple[DesiredPartition, ...],
    staging: tuple[StagingObservation, ...],
    targets: tuple[TargetObservation, ...],
) -> tuple[PartitionDecision, ...]:
    """Validate one snapshot fully, then classify every desired partition."""
    checked_desired = _checked_desired(desired)
    desired_ids = {item.partition_id for item in checked_desired}
    checked_staging = _checked_observations(
        staging,
        record_type=StagingObservation,
        desired_ids=desired_ids,
        field="staging",
    )
    checked_targets = _checked_observations(
        targets,
        record_type=TargetObservation,
        desired_ids=desired_ids,
        field="targets",
    )
    staging_by_id = {item.partition_id: item for item in checked_staging}
    target_by_id = {item.partition_id: item for item in checked_targets}
    return tuple(
        _decision(
            item,
            staging_by_id.get(item.partition_id),
            target_by_id.get(item.partition_id),
        )
        for item in checked_desired
    )
```

## Example

```python
def content(marker: str, rows: int) -> PartitionContent:
    return PartitionContent("records-v1", marker * 64, rows, rows * 20)


desired = (
    DesiredPartition("p-001", content("a", 4), TargetStamp(10, "t-001")),
    DesiredPartition("p-002", content("b", 5), TargetStamp(20, "t-002")),
    DesiredPartition("p-003", content("c", 6), TargetStamp(30, "t-003")),
    DesiredPartition("p-004", content("d", 7), TargetStamp(40, "t-004")),
)
staging = (
    StagingObservation("p-002", content("b", 5)),
    StagingObservation("p-003", content("c", 6)),
)
targets = (
    TargetObservation("p-001", content("a", 4), TargetStamp(11, "new-001")),
    TargetObservation("p-002", content("b", 5), TargetStamp(21, "new-002")),
    TargetObservation("p-003", content("e", 2), TargetStamp(30, "t-003")),
)

plan = plan_partition_publication(desired, staging, targets)

assert tuple(item.action for item in plan) == (
    PublicationAction.ALREADY_CURRENT,
    PublicationAction.CLEANUP_CANDIDATE,
    PublicationAction.PROMOTE_CANDIDATE,
    PublicationAction.BLOCKED,
)
assert tuple(item.partition_id for item in plan) == (
    "p-001",
    "p-002",
    "p-003",
    "p-004",
)
assert plan[2].observed_target == TargetStamp(30, "t-003")
assert "missing" in plan[3].reason
```

## Trade-offs and Limitations

Preflight takes linear time and memory for at most 256 desired partitions and
256 observations of each kind. IDs, tokens, digests, row counts, byte counts,
tuple types, record types, uniqueness, order, and references are validated
before classification begins. Representational errors raise `TypeError` or
`ValueError` and return no plan; valid but incomplete or conflicting evidence
produces `BLOCKED` beside any independently actionable partitions.

The conservative identifier syntax, SHA-256 shape check, fixed count range,
and exact metadata equality reject some legitimate systems. A digest is not
recomputed or authenticated, and observations are only as trustworthy and
fresh as their producer. There is deliberately no batch success flag: partial
progress remains visible, but no multi-partition atomicity is implied.

The function performs no SQL or database call, transaction, promotion,
cleanup, retry, locking, ownership check, compare-and-swap, or I/O. Candidate
means only that this snapshot is coherent. An external executor owns
conditional revalidation, one-partition execution, and post-action
reobservation, including resolution of ambiguous outcomes before any retry.

## Related Snippets

<!-- catalog:related:start -->
- [Validate a Bounded Chunk Manifest Before a Conditional Version Switch](validate-a-bounded-chunk-manifest-before-a-conditional-version-switch.md)
- [Validate a Bounded Stage-Verify-Pointer-Switch Log](validate-a-bounded-stage-verify-pointer-switch-log.md)
- [Compare and Swap a Versioned SQLite Setting with One Conditional Update](compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md)
<!-- catalog:related:end -->
