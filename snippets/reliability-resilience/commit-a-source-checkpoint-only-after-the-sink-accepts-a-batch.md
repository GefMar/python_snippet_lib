---
title: "Commit a Source Checkpoint Only After the Sink Accepts a Batch"
snippet_type: pattern
use_cases:
  - retry-recovery
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - retry-only-eligible-items-in-a-bounded-batch.md
  - compensate-completed-workflow-steps-in-reverse-order.md
  - ../storage-databases/build-and-apply-a-deterministic-mapping-delta.md
---

# Commit a Source Checkpoint Only After the Sink Accepts a Batch

## Idea and Problem

Advance a source checkpoint only after the sink has durably accepted every prepared event from that exact bounded batch.

Committing first can lose data when the sink fails. Sending first preserves
at-least-once delivery: preparation or sink failure leaves the checkpoint
unchanged, while a failure after sink acceptance but before checkpoint commit
causes the same source batch to be delivered again.

## When to Use

Use this pattern when the source attaches an opaque next checkpoint to each
finite batch and the sink exposes an all-or-error durable write. Prepared
events need stable IDs, and the sink should deduplicate those IDs or otherwise
make repeated delivery harmless. Give one worker exclusive ownership of the
source partition, or make `commit_checkpoint` compare and set the expected
previous position.

Keep reading, restart loops, retry timing, and partition assignment outside the
one-batch function. A sink with partial or ambiguous acknowledgement needs a
stronger reconciliation protocol before any checkpoint can be committed.

## Implementation

```python
import re
from collections.abc import Callable
from dataclasses import dataclass


_MAX_SOURCE_RECORDS = 1_000
_MAX_RECORD_BYTES = 256 * 1024
_MAX_EVENT_BYTES = 256 * 1024
_MAX_CHECKPOINT_BYTES = 256
_OPAQUE_ID = re.compile(r"[A-Za-z0-9][A-Za-z0-9._:-]{0,127}", re.ASCII)


@dataclass(frozen=True, slots=True)
class SourceBatch:
    next_checkpoint: bytes
    records: tuple[bytes, ...]


@dataclass(frozen=True, slots=True)
class PreparedEvent:
    event_id: str
    payload: bytes


@dataclass(frozen=True, slots=True)
class BatchDelivery:
    checkpoint: bytes
    source_records: int
    delivered_events: int


def _validate_source_batch(batch: object) -> SourceBatch:
    if not isinstance(batch, SourceBatch):
        raise TypeError("batch must be a SourceBatch")
    if not isinstance(batch.next_checkpoint, bytes):
        raise TypeError("next checkpoint must be immutable bytes")
    if not 1 <= len(batch.next_checkpoint) <= _MAX_CHECKPOINT_BYTES:
        raise ValueError("next checkpoint length is outside the supported range")
    if not isinstance(batch.records, tuple):
        raise TypeError("source records must be a tuple")
    if len(batch.records) > _MAX_SOURCE_RECORDS:
        raise ValueError("source batch exceeds the supported record count")
    for record in batch.records:
        if not isinstance(record, bytes):
            raise TypeError("source records must be immutable bytes")
        if len(record) > _MAX_RECORD_BYTES:
            raise ValueError("a source record exceeds the supported size")
    return batch


def _validate_event(event: object) -> PreparedEvent:
    if not isinstance(event, PreparedEvent):
        raise TypeError("prepare must return PreparedEvent or None")
    if _OPAQUE_ID.fullmatch(event.event_id) is None:
        raise ValueError("event_id has an invalid format")
    if not isinstance(event.payload, bytes):
        raise TypeError("event payloads must be immutable bytes")
    if len(event.payload) > _MAX_EVENT_BYTES:
        raise ValueError("an event payload exceeds the supported size")
    return event


def deliver_source_batch(
    batch: SourceBatch,
    *,
    prepare: Callable[[bytes], PreparedEvent | None],
    write_all: Callable[
        [tuple[PreparedEvent, ...]],
        None,
    ],
    commit_checkpoint: Callable[
        [bytes],
        None,
    ],
) -> BatchDelivery:
    validated_batch = _validate_source_batch(batch)
    if not callable(prepare) or not callable(write_all) or not callable(
        commit_checkpoint
    ):
        raise TypeError("prepare, write_all, and commit_checkpoint must be callable")

    prepared = []
    seen_event_ids = set()
    for record in validated_batch.records:
        candidate = prepare(record)
        if candidate is None:
            continue
        event = _validate_event(candidate)
        if event.event_id in seen_event_ids:
            raise ValueError("prepared event IDs must be unique within the batch")
        seen_event_ids.add(event.event_id)
        prepared.append(event)

    events = tuple(prepared)
    if events:
        write_all(events)
    commit_checkpoint(validated_batch.next_checkpoint)
    return BatchDelivery(
        validated_batch.next_checkpoint,
        len(validated_batch.records),
        len(events),
    )
```

## Example

```python
accepted_batches = []
committed = []


def prepare_record(record: bytes) -> PreparedEvent | None:
    if record == b"skip":
        return None
    event_id, payload = record.split(b"|", maxsplit=1)
    return PreparedEvent(event_id.decode("ascii"), payload)


def accept_all(events: tuple[PreparedEvent, ...]) -> None:
    accepted_batches.append(events)


delivery = deliver_source_batch(
    SourceBatch(b"position-9", (b"event-7|alpha", b"skip", b"event-8|beta")),
    prepare=prepare_record,
    write_all=accept_all,
    commit_checkpoint=committed.append,
)

before_failure = tuple(committed)


def reject_all(events: tuple[PreparedEvent, ...]) -> None:
    raise OSError("sink did not acknowledge the batch")


try:
    deliver_source_batch(
        SourceBatch(b"position-10", (b"event-9|gamma",)),
        prepare=prepare_record,
        write_all=reject_all,
        commit_checkpoint=committed.append,
    )
except OSError:
    sink_failure_propagated = True
else:
    sink_failure_propagated = False

empty_delivery = deliver_source_batch(
    SourceBatch(b"position-11", (b"skip",)),
    prepare=prepare_record,
    write_all=accept_all,
    commit_checkpoint=committed.append,
)

assert (
    delivery,
    tuple(event.event_id for event in accepted_batches[0]),
    before_failure,
    sink_failure_propagated,
    empty_delivery.delivered_events,
    tuple(committed),
) == (
    BatchDelivery(b"position-9", 3, 2),
    ("event-7", "event-8"),
    (b"position-9",),
    True,
    0,
    (b"position-9", b"position-11"),
)
```

## Trade-offs and Limitations

This ordering provides at-least-once delivery, not exactly-once delivery. If
`write_all` succeeds and the process stops or `commit_checkpoint` fails, the
same stable event IDs can reach the sink again. The function deliberately does
not attempt rollback, restart, or hidden retries because it cannot know whether
an ambiguous external write was durable.

The source batch and every record are bounded and materialized, and prepared
events temporarily require comparable additional memory. `write_all` must mean
durable all-or-error acceptance; a function that returns after buffering
locally or partially accepting events does not satisfy the contract. Filtering
all records still commits the batch's attached checkpoint because the source
records were read and classified successfully. Never replace that checkpoint
with a separately queried later cursor. Keep `prepare` deterministic and free
of external side effects because any failure before commit can repeat it.

## Related Snippets

<!-- catalog:related:start -->
- [Retry Only Eligible Items in a Bounded Batch](retry-only-eligible-items-in-a-bounded-batch.md)
- [Compensate Completed Workflow Steps in Reverse Order](compensate-completed-workflow-steps-in-reverse-order.md)
- [Build and Apply a Deterministic Mapping Delta](../storage-databases/build-and-apply-a-deterministic-mapping-delta.md)
<!-- catalog:related:end -->
