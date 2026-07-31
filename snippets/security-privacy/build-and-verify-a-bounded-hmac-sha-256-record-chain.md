---
title: "Build and Verify a Bounded HMAC-SHA-256 Record Chain"
snippet_type: pattern
use_cases:
  - security
  - serialization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - authenticate-bounded-payloads-with-versioned-hmac-keys.md
  - build-and-verify-a-bounded-sha-256-merkle-inclusion-proof.md
  - verify-a-bounded-byte-stream-before-returning-its-payload.md
---

# Build and Verify a Bounded HMAC-SHA-256 Record Chain

## Idea and Problem

Authenticate an ordered segment of byte records by making every HMAC depend on the preceding trusted tag and its absolute record index.

The fixed binary frame also covers a versioned profile domain, chain ID,
segment record count and payload length. This separates chains and purposes,
makes byte boundaries unambiguous, and causes deletion, insertion, reordering
or mutation within a presented segment to fail. An independently trusted
terminal checkpoint is required so that presenting an older valid prefix
cannot silently pass as the complete expected segment.

## When to Use

Use this closed profile when one serialized writer and its verifier share a
secret key, records are already represented as bounded immutable bytes, and a
trusted store can retain both the starting and expected terminal checkpoints.
It is useful for detecting accidental or unauthorized changes to a bounded
ordered export or append segment before consuming its records.

Use an established authenticated container or protocol when data crosses
independent systems. Use authenticated encryption when payload secrecy is also
required, and use digital signatures when verifiers must not possess a key
that can rewrite the chain. Persist checkpoints in storage with rollback and
concurrency guarantees at least as strong as the records they protect.

## Implementation

```python
import hmac
from collections.abc import Callable
from dataclasses import dataclass

_CHAIN_DOMAIN = b"bounded-hmac-sha256-record-chain-v1\x00"
_MIN_CHAIN_KEY_BYTES = 32
_MAX_CHAIN_KEY_BYTES = 1024
_CHAIN_ID_BYTES = 16
_TAG_BYTES = 32
_MAX_CHAIN_RECORDS = 1024
_MAX_RECORD_BYTES = 64 * 1024
_MAX_CHAIN_PAYLOAD_BYTES = 4 * 1024 * 1024
_MAX_UINT64 = (1 << 64) - 1


@dataclass(frozen=True, slots=True)
class ChainCheckpoint:
    next_index: int
    previous_tag: bytes


@dataclass(frozen=True, slots=True)
class AuthenticatedRecord:
    index: int
    payload: bytes
    tag: bytes


def _validate_chain_inputs(key: bytes, chain_id: bytes) -> None:
    if type(key) is not bytes:
        raise TypeError("key must be exact immutable bytes")
    if not _MIN_CHAIN_KEY_BYTES <= len(key) <= _MAX_CHAIN_KEY_BYTES:
        raise ValueError("key length is outside the supported range")
    if type(chain_id) is not bytes:
        raise TypeError("chain_id must be exact immutable bytes")
    if len(chain_id) != _CHAIN_ID_BYTES:
        raise ValueError("chain_id must contain exactly 16 bytes")


def _validate_checkpoint(checkpoint: ChainCheckpoint, name: str) -> None:
    if type(checkpoint) is not ChainCheckpoint:
        raise TypeError(f"{name} must be an exact ChainCheckpoint")
    if type(checkpoint.next_index) is not int:
        raise TypeError(f"{name}.next_index must be an exact integer")
    if not 0 <= checkpoint.next_index <= _MAX_UINT64:
        raise ValueError(f"{name}.next_index is outside unsigned 64-bit range")
    if type(checkpoint.previous_tag) is not bytes:
        raise TypeError(f"{name}.previous_tag must be exact immutable bytes")
    if len(checkpoint.previous_tag) != _TAG_BYTES:
        raise ValueError(f"{name}.previous_tag must contain exactly 32 bytes")


def _record_tag(
    key: bytes,
    chain_id: bytes,
    segment_record_count: int,
    index: int,
    previous_tag: bytes,
    payload: bytes,
) -> bytes:
    frame = b"".join(
        (
            _CHAIN_DOMAIN,
            chain_id,
            segment_record_count.to_bytes(4, "big"),
            index.to_bytes(8, "big"),
            previous_tag,
            len(payload).to_bytes(4, "big"),
            payload,
        )
    )
    return hmac.digest(key, frame, "sha256")


def _validate_payloads(payloads: tuple[bytes, ...]) -> None:
    if type(payloads) is not tuple:
        raise TypeError("payloads must be an exact tuple")
    if len(payloads) > _MAX_CHAIN_RECORDS:
        raise ValueError("record count exceeds the supported limit")
    total_bytes = 0
    for payload_index, payload in enumerate(payloads):
        if type(payload) is not bytes:
            raise TypeError(f"payloads[{payload_index}] must be exact immutable bytes")
        if len(payload) > _MAX_RECORD_BYTES:
            raise ValueError(f"payloads[{payload_index}] exceeds the per-record limit")
        total_bytes += len(payload)
        if total_bytes > _MAX_CHAIN_PAYLOAD_BYTES:
            raise ValueError("aggregate payload exceeds the supported limit")


def build_hmac_record_chain(
    key: bytes,
    chain_id: bytes,
    checkpoint: ChainCheckpoint,
    payloads: tuple[bytes, ...],
) -> tuple[tuple[AuthenticatedRecord, ...], ChainCheckpoint]:
    """Build one bounded authenticated segment from a trusted checkpoint."""
    _validate_chain_inputs(key, chain_id)
    _validate_checkpoint(checkpoint, "checkpoint")
    _validate_payloads(payloads)
    if checkpoint.next_index + len(payloads) > _MAX_UINT64:
        raise ValueError("terminal next_index would exceed unsigned 64-bit range")

    records: list[AuthenticatedRecord] = []
    previous_tag = checkpoint.previous_tag
    for offset, payload in enumerate(payloads):
        index = checkpoint.next_index + offset
        tag = _record_tag(
            key,
            chain_id,
            len(payloads),
            index,
            previous_tag,
            payload,
        )
        records.append(AuthenticatedRecord(index, payload, tag))
        previous_tag = tag

    terminal = ChainCheckpoint(checkpoint.next_index + len(payloads), previous_tag)
    return tuple(records), terminal


def verify_hmac_record_chain(
    key: bytes,
    chain_id: bytes,
    checkpoint: ChainCheckpoint,
    records: tuple[AuthenticatedRecord, ...],
    expected_terminal: ChainCheckpoint,
) -> ChainCheckpoint:
    """Verify one complete expected segment and return its terminal checkpoint."""
    _validate_chain_inputs(key, chain_id)
    _validate_checkpoint(checkpoint, "checkpoint")
    _validate_checkpoint(expected_terminal, "expected_terminal")
    if type(records) is not tuple:
        raise TypeError("records must be an exact tuple")
    if len(records) > _MAX_CHAIN_RECORDS:
        raise ValueError("record count exceeds the supported limit")
    if checkpoint.next_index + len(records) > _MAX_UINT64:
        raise ValueError("terminal next_index would exceed unsigned 64-bit range")

    total_bytes = 0
    for record_offset, record in enumerate(records):
        if type(record) is not AuthenticatedRecord:
            raise TypeError(f"records[{record_offset}] must be an exact AuthenticatedRecord")
        if type(record.index) is not int:
            raise TypeError(f"records[{record_offset}].index must be an exact integer")
        if type(record.payload) is not bytes:
            raise TypeError(f"records[{record_offset}].payload must be exact immutable bytes")
        if type(record.tag) is not bytes:
            raise TypeError(f"records[{record_offset}].tag must be exact immutable bytes")
        if len(record.payload) > _MAX_RECORD_BYTES:
            raise ValueError(f"records[{record_offset}].payload exceeds the per-record limit")
        if len(record.tag) != _TAG_BYTES:
            raise ValueError(f"records[{record_offset}].tag must contain exactly 32 bytes")
        total_bytes += len(record.payload)
        if total_bytes > _MAX_CHAIN_PAYLOAD_BYTES:
            raise ValueError("aggregate payload exceeds the supported limit")

    previous_tag = checkpoint.previous_tag
    for record_offset, record in enumerate(records):
        expected_index = checkpoint.next_index + record_offset
        if record.index != expected_index:
            raise ValueError(f"records[{record_offset}] has a non-sequential index")
        expected_tag = _record_tag(
            key,
            chain_id,
            len(records),
            expected_index,
            previous_tag,
            record.payload,
        )
        if not hmac.compare_digest(expected_tag, record.tag):
            raise ValueError(f"records[{record_offset}] has an invalid tag")
        previous_tag = expected_tag

    terminal = ChainCheckpoint(checkpoint.next_index + len(records), previous_tag)
    if terminal.next_index != expected_terminal.next_index or not hmac.compare_digest(
        terminal.previous_tag,
        expected_terminal.previous_tag,
    ):
        raise ValueError("terminal checkpoint does not match the trusted expectation")
    return terminal
```

## Example

```python
def rejected(operation: Callable[[], object]) -> bool:
    try:
        operation()
    except (TypeError, ValueError):
        return True
    return False


key = b"k" * 32  # Deterministic demonstration key, not a production secret.
chain_id = bytes.fromhex("00112233445566778899aabbccddeeff")
starting = ChainCheckpoint(7, bytes(range(32)))
payloads = (b"alpha", b"", b"\x00beta")
records, terminal = build_hmac_record_chain(key, chain_id, starting, payloads)

# Assemble the first frame independently of the implementation helper.
first_frame = b"".join(
    (
        b"bounded-hmac-sha256-record-chain-v1\x00",
        chain_id,
        (3).to_bytes(4, "big"),
        (7).to_bytes(8, "big"),
        bytes(range(32)),
        (5).to_bytes(4, "big"),
        b"alpha",
    )
)
assert records[0].tag == hmac.digest(key, first_frame, "sha256")
assert verify_hmac_record_chain(key, chain_id, starting, records, terminal) == terminal

changed_payload = (
    AuthenticatedRecord(records[0].index, b"Alpha", records[0].tag),
    *records[1:],
)
changed_tag = (
    *records[:-1],
    AuthenticatedRecord(records[-1].index, records[-1].payload, b"!" * 32),
)
changed_index = (
    AuthenticatedRecord(records[0].index + 1, records[0].payload, records[0].tag),
    *records[1:],
)
wrong_terminal = ChainCheckpoint(terminal.next_index, b"?" * 32)

assert all(
    (
        rejected(lambda: verify_hmac_record_chain(key, chain_id, starting, records[:-1], terminal)),
        rejected(
            lambda: verify_hmac_record_chain(
                key, chain_id, starting, records[:1] + records[2:], terminal
            )
        ),
        rejected(
            lambda: verify_hmac_record_chain(
                key, chain_id, starting, tuple(reversed(records)), terminal
            )
        ),
        rejected(
            lambda: verify_hmac_record_chain(
                key, chain_id, starting, records[:1] + records[:1] + records[2:], terminal
            )
        ),
        rejected(
            lambda: verify_hmac_record_chain(key, chain_id, starting, changed_payload, terminal)
        ),
        rejected(lambda: verify_hmac_record_chain(key, chain_id, starting, changed_tag, terminal)),
        rejected(
            lambda: verify_hmac_record_chain(key, chain_id, starting, changed_index, terminal)
        ),
        rejected(
            lambda: verify_hmac_record_chain(b"z" * 32, chain_id, starting, records, terminal)
        ),
        rejected(lambda: verify_hmac_record_chain(key, b"x" * 16, starting, records, terminal)),
        rejected(lambda: build_hmac_record_chain(b"k" * 31, chain_id, starting, ())),
        rejected(lambda: build_hmac_record_chain(b"k" * 1025, chain_id, starting, ())),
        rejected(lambda: build_hmac_record_chain(key, b"i" * 15, starting, ())),
        rejected(lambda: build_hmac_record_chain(key, b"i" * 17, starting, ())),
        rejected(
            lambda: verify_hmac_record_chain(
                key, chain_id, ChainCheckpoint(8, starting.previous_tag), records, terminal
            )
        ),
        rejected(
            lambda: verify_hmac_record_chain(key, chain_id, starting, records, wrong_terminal)
        ),
        rejected(
            lambda: build_hmac_record_chain(
                key, chain_id, starting, (b"x" * (_MAX_RECORD_BYTES + 1),)
            )
        ),
        rejected(
            lambda: build_hmac_record_chain(
                key, chain_id, starting, (b"",) * (_MAX_CHAIN_RECORDS + 1)
            )
        ),
        rejected(
            lambda: build_hmac_record_chain(
                key, chain_id, starting, (b"x" * _MAX_RECORD_BYTES,) * 65
            )
        ),
        rejected(
            lambda: build_hmac_record_chain(
                key, chain_id, ChainCheckpoint(_MAX_UINT64, b"0" * 32), (b"x",)
            )
        ),
    )
)

empty_records, unchanged = build_hmac_record_chain(key, chain_id, terminal, ())
assert empty_records == ()
assert verify_hmac_record_chain(key, chain_id, terminal, empty_records, unchanged) == terminal

maximum_payload = b"x" * _MAX_RECORD_BYTES
accepted_boundaries = (
    (b"q" * _MAX_CHAIN_KEY_BYTES, (maximum_payload,)),
    (key, (b"",) * _MAX_CHAIN_RECORDS),
    (key, (maximum_payload,) * 64),
)
boundary_round_trips: list[bool] = []
for accepted_key, accepted_payloads in accepted_boundaries:
    accepted_records, accepted_terminal = build_hmac_record_chain(
        accepted_key,
        chain_id,
        starting,
        accepted_payloads,
    )
    boundary_round_trips.append(
        verify_hmac_record_chain(
            accepted_key,
            chain_id,
            starting,
            accepted_records,
            accepted_terminal,
        )
        == accepted_terminal
    )
assert all(boundary_round_trips)
```

## Trade-offs and Limitations

Building and verification are linear in the total payload size and retain the
bounded record tuple in memory. Every tag adds 32 bytes. Including the segment
record count intentionally binds tags to one presented segment; extending the
chain requires building a new segment from the previously trusted terminal
checkpoint rather than appending records to an already authenticated tuple.

This is a private closed profile, not an interoperable log format. It does not
encrypt payloads or provide non-repudiation: anyone holding the shared key can
rewrite a chain. Two writers can also create different valid continuations
from one checkpoint, so writes must be serialized. Tail deletion and rollback
remain detectable only while the expected terminal checkpoint is stored and
retrieved independently from storage that an attacker cannot roll back with
the records. Key generation, rotation, erasure and durable checkpoint storage
are deliberately outside this focused primitive.

## Related Snippets

<!-- catalog:related:start -->
- [Authenticate Bounded Payloads with Versioned HMAC Keys](authenticate-bounded-payloads-with-versioned-hmac-keys.md)
- [Build and Verify a Bounded SHA-256 Merkle Inclusion Proof](build-and-verify-a-bounded-sha-256-merkle-inclusion-proof.md)
- [Verify a Bounded Byte Stream Before Returning Its Payload](verify-a-bounded-byte-stream-before-returning-its-payload.md)
<!-- catalog:related:end -->
