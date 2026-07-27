---
title: "Validate a Bounded Stage-Verify-Pointer-Switch Log"
snippet_type: pattern
use_cases:
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - decide-whether-to-restore-a-versioned-snapshot.md
  - compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md
  - ../reliability-resilience/plan-remaining-stages-from-a-validated-completed-prefix.md
---

# Validate a Bounded Stage-Verify-Pointer-Switch Log

## Idea and Problem

Reduce a bounded stage-verify-activate log only when every version follows the required order and every pointer expectation is current.

Staging binds a version to one digest, verification records one explicit final
outcome for that same digest, and activation changes the modeled pointer in one
reducer step. An expected-active field rejects a transition based on stale
state, while a frozen trace makes every accepted step reproducible.

## When to Use

Use this pattern to validate an already collected finite transition log before
another layer interprets its final state. Version IDs and lowercase SHA-256
digests must already be available as trusted immutable values. The reducer is
useful for checking order and optimistic pointer expectations independently of
any storage engine.

Keep data movement, persistent compare-and-swap, synchronization, and recovery
in an execution layer with semantics appropriate to the actual storage system.

## Implementation

```python
import re
from dataclasses import dataclass
from typing import Literal


_MAX_EVENTS = 256
_MAX_VERSIONS = 64
_VERSION_ID = re.compile(r"[a-z][a-z0-9._:-]{0,63}", re.ASCII)
_LOWER_SHA256 = re.compile(r"[0-9a-f]{64}", re.ASCII)


@dataclass(frozen=True, slots=True)
class StageVersion:
    version: str
    digest: str


@dataclass(frozen=True, slots=True)
class VerifyVersion:
    version: str
    digest: str
    passed: bool


@dataclass(frozen=True, slots=True)
class ActivateVersion:
    version: str
    expected_active: str


StageLogEvent = StageVersion | VerifyVersion | ActivateVersion


@dataclass(frozen=True, slots=True)
class StagedVersionState:
    version: str
    digest: str
    verification: Literal["pending", "passed", "failed"]
    was_activated: bool


@dataclass(frozen=True, slots=True)
class AcceptedTransition:
    event: StageLogEvent
    active_after: str


@dataclass(frozen=True, slots=True)
class StageLogResult:
    active_version: str
    staged_versions: tuple[StagedVersionState, ...]
    accepted_trace: tuple[AcceptedTransition, ...]


def _validated_version(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if _VERSION_ID.fullmatch(value) is None:
        raise ValueError(f"{field} must be a 1-64 byte conservative ASCII ID")
    return value


def _validated_digest(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if _LOWER_SHA256.fullmatch(value) is None:
        raise ValueError(f"{field} must be a lowercase SHA-256 digest")
    return value


def _validated_event(value: object, *, index: int) -> StageLogEvent:
    field = f"events[{index}]"
    if type(value) is StageVersion:
        return StageVersion(
            _validated_version(value.version, field=f"{field}.version"),
            _validated_digest(value.digest, field=f"{field}.digest"),
        )
    if type(value) is VerifyVersion:
        version = _validated_version(value.version, field=f"{field}.version")
        digest = _validated_digest(value.digest, field=f"{field}.digest")
        if type(value.passed) is not bool:
            raise TypeError(f"{field}.passed must be an exact boolean")
        return VerifyVersion(version, digest, value.passed)
    if type(value) is ActivateVersion:
        return ActivateVersion(
            _validated_version(value.version, field=f"{field}.version"),
            _validated_version(
                value.expected_active,
                field=f"{field}.expected_active",
            ),
        )
    raise TypeError(f"{field} has an unsupported exact event type")


def _validated_events(
    value: object,
    *,
    initial_active: str,
) -> tuple[StageLogEvent, ...]:
    if type(value) is not tuple:
        raise TypeError("events must be an exact tuple")
    if len(value) > _MAX_EVENTS:
        raise ValueError(f"events must contain at most {_MAX_EVENTS} entries")

    events = tuple(
        _validated_event(event, index=index)
        for index, event in enumerate(value)
    )
    referenced_versions = {initial_active}
    for event in events:
        referenced_versions.add(event.version)
        if type(event) is ActivateVersion:
            referenced_versions.add(event.expected_active)
    if len(referenced_versions) > _MAX_VERSIONS:
        raise ValueError(
            f"the log must refer to at most {_MAX_VERSIONS} versions"
        )
    return events


def validate_stage_log(
    initial_active: str,
    events: tuple[StageLogEvent, ...],
) -> StageLogResult:
    """Validate and reduce an immutable stage-verify-activate event log."""
    active = _validated_version(initial_active, field="initial_active")
    validated_events = _validated_events(events, initial_active=active)

    states: dict[str, StagedVersionState] = {}
    staging_order: list[str] = []
    trace: list[AcceptedTransition] = []

    for index, event in enumerate(validated_events):
        if type(event) is StageVersion:
            if event.version == initial_active or event.version in states:
                raise ValueError(f"events[{index}] stages a duplicate version")
            states[event.version] = StagedVersionState(
                event.version,
                event.digest,
                "pending",
                False,
            )
            staging_order.append(event.version)

        elif type(event) is VerifyVersion:
            state = states.get(event.version)
            if state is None:
                raise ValueError(
                    f"events[{index}] verifies a version that was not staged"
                )
            if event.digest != state.digest:
                raise ValueError(f"events[{index}] has a digest mismatch")
            if state.verification != "pending":
                raise ValueError(f"events[{index}] repeats verification")
            states[event.version] = StagedVersionState(
                state.version,
                state.digest,
                "passed" if event.passed else "failed",
                state.was_activated,
            )

        else:
            state = states.get(event.version)
            if state is None:
                raise ValueError(
                    f"events[{index}] activates a version that was not staged"
                )
            if event.expected_active != active:
                raise ValueError(f"events[{index}] has a stale expected pointer")
            if state.verification != "passed":
                raise ValueError(
                    f"events[{index}] activates a version without successful verification"
                )
            if state.was_activated:
                raise ValueError(f"events[{index}] repeats activation")
            active = event.version
            states[event.version] = StagedVersionState(
                state.version,
                state.digest,
                state.verification,
                True,
            )

        trace.append(AcceptedTransition(event, active))

    return StageLogResult(
        active_version=active,
        staged_versions=tuple(states[version] for version in staging_order),
        accepted_trace=tuple(trace),
    )
```

## Example

```python
first_digest = "a" * 64
second_digest = "b" * 64
events = (
    StageVersion("release-8", first_digest),
    VerifyVersion("release-8", first_digest, passed=True),
    ActivateVersion("release-8", expected_active="release-7"),
    StageVersion("release-9", second_digest),
    VerifyVersion("release-9", second_digest, passed=False),
)

result = validate_stage_log("release-7", events)

assert result == StageLogResult(
    active_version="release-8",
    staged_versions=(
        StagedVersionState("release-8", first_digest, "passed", True),
        StagedVersionState("release-9", second_digest, "failed", False),
    ),
    accepted_trace=(
        AcceptedTransition(events[0], "release-7"),
        AcceptedTransition(events[1], "release-7"),
        AcceptedTransition(events[2], "release-8"),
        AcceptedTransition(events[3], "release-8"),
        AcceptedTransition(events[4], "release-8"),
    ),
)
```

## Trade-offs and Limitations

The reducer uses linear time and memory for at most 256 events referring to at
most 64 versions. It validates every event's exact type and bounded fields
before applying semantic transitions. A failed verification is final, a
version cannot be restaged or reverified, and an activated version cannot be
activated again; recovery or rollback requires a separately modeled version.

Digest validation checks lowercase SHA-256 shape only; it neither hashes nor
authenticates content. This pure reducer does not copy data, connect to SQL or
SQLite, access providers, create tables or views, perform I/O, start a
transaction, persist or synchronize the pointer, guarantee durability or
storage atomicity, coordinate concurrent writers, clean up, retry, or roll
back. Activation is one in-memory reducer step, not a claim about an external
storage transition.

## Related Snippets

<!-- catalog:related:start -->
- [Decide Whether to Restore a Versioned Snapshot](decide-whether-to-restore-a-versioned-snapshot.md)
- [Compare and Swap a Versioned SQLite Setting with One Conditional Update](compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md)
- [Plan Remaining Stages from a Validated Completed Prefix](../reliability-resilience/plan-remaining-stages-from-a-validated-completed-prefix.md)
<!-- catalog:related:end -->
