---
title: "Plan Remaining Stages from a Validated Completed Prefix"
snippet_type: algorithm
use_cases:
  - retry-recovery
  - automation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - commit-a-source-checkpoint-only-after-the-sink-accepts-a-batch.md
  - ../data-processing/run-a-pipeline-with-lazy-conversion-between-two-views.md
  - ../storage-databases/decide-whether-to-restore-a-versioned-snapshot.md
---

# Plan Remaining Stages from a Validated Completed Prefix

## Idea and Problem

Derive an immutable resume decision only when recorded completion is an exact prefix of the current ordered stage plan.

A stale, reordered, or unfamiliar completion history must not be treated as progress. Validating the complete plan and its recorded prefix before selecting the next stage makes that boundary explicit and keeps the decision deterministic.

## When to Use

Use this algorithm when a caller already has a canonical ordered plan and a separately loaded completion record. Stage identifiers must be stable conservative ASCII tokens, and changing the order or inserting an earlier stage must deliberately invalidate an old record. Handle plan-version migration outside this function.

## Implementation

```python
import re
from dataclasses import dataclass


_MAX_STAGES = 256
_MAX_STAGE_ID_BYTES = 64
_MAX_STAGE_ID_BYTES_PER_TUPLE = 16_384
_STAGE_ID_PATTERN = re.compile(r"[A-Za-z0-9][A-Za-z0-9._:-]{0,63}", re.ASCII)


@dataclass(frozen=True, slots=True)
class StageResumeDecision:
    completed: tuple[str, ...]
    remaining: tuple[str, ...]
    next_stage: str | None


def _validate_stage_ids(
    value: object,
    *,
    name: str,
    allow_empty: bool,
) -> tuple[str, ...]:
    if type(value) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if not allow_empty and not value:
        raise ValueError(f"{name} must not be empty")
    if len(value) > _MAX_STAGES:
        raise ValueError(f"{name} must contain at most {_MAX_STAGES} identifiers")

    validated: list[str] = []
    seen: set[str] = set()
    total_bytes = 0

    for index, stage_id in enumerate(value):
        if type(stage_id) is not str:
            raise TypeError(f"{name}[{index}] must be an exact string")
        if _STAGE_ID_PATTERN.fullmatch(stage_id) is None:
            raise ValueError(
                f"{name}[{index}] must be a 1-{_MAX_STAGE_ID_BYTES} byte ASCII token"
            )
        if stage_id in seen:
            raise ValueError(f"{name} contains duplicate identifier {stage_id!r}")

        total_bytes += len(stage_id.encode("ascii"))
        if total_bytes > _MAX_STAGE_ID_BYTES_PER_TUPLE:
            raise ValueError(
                f"{name} identifiers exceed {_MAX_STAGE_ID_BYTES_PER_TUPLE} bytes"
            )

        seen.add(stage_id)
        validated.append(stage_id)

    return tuple(validated)


def plan_remaining_stages(
    stages: tuple[str, ...],
    completed: tuple[str, ...],
) -> StageResumeDecision:
    """Validate recorded progress and derive the ordered work that remains."""
    validated_stages = _validate_stage_ids(
        stages,
        name="stages",
        allow_empty=False,
    )
    validated_completed = _validate_stage_ids(
        completed,
        name="completed",
        allow_empty=True,
    )

    expected_prefix = validated_stages[: len(validated_completed)]
    if validated_completed != expected_prefix:
        raise ValueError("completed must be an exact prefix of stages")

    remaining = validated_stages[len(validated_completed) :]
    next_stage = remaining[0] if remaining else None
    return StageResumeDecision(
        completed=validated_completed,
        remaining=remaining,
        next_stage=next_stage,
    )
```

## Example

```python
release_stages = ("inspect", "normalize", "publish")
recorded_completion = ("inspect", "normalize")

decision = plan_remaining_stages(release_stages, recorded_completion)

assert decision == StageResumeDecision(
    completed=("inspect", "normalize"),
    remaining=("publish",),
    next_stage="publish",
)
```

## Trade-offs and Limitations

Validation is linear in the number of stages and materializes bounded tuples and sets. Any change before the recorded prefix intentionally rejects that history; systems that evolve plans need an explicit versioning and migration policy. The decision does not prove that completed stages produced valid artifacts or that future actions are idempotent. It performs no persistence, retries, callbacks, concurrency control, or I/O.

## Related Snippets

<!-- catalog:related:start -->
- [Commit a Source Checkpoint Only After the Sink Accepts a Batch](commit-a-source-checkpoint-only-after-the-sink-accepts-a-batch.md)
- [Run a Pipeline with Lazy Conversion Between Two Views](../data-processing/run-a-pipeline-with-lazy-conversion-between-two-views.md)
- [Decide Whether to Restore a Versioned Snapshot](../storage-databases/decide-whether-to-restore-a-versioned-snapshot.md)
<!-- catalog:related:end -->
