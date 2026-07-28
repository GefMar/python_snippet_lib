---
title: "Validate a Bounded Chunk Manifest Before a Conditional Version Switch"
snippet_type: pattern
use_cases:
  - concurrency-control
  - persistence
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - validate-a-bounded-stage-verify-pointer-switch-log.md
  - fingerprint-a-bounded-flat-file-set-with-framed-sha-256.md
  - compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md
---

# Validate a Bounded Chunk Manifest Before a Conditional Version Switch

## Idea and Problem

Validate every staged chunk against a frozen manifest before planning a version-pointer switch that still depends on an expected current value.

The expected manifest fixes declaration order, counts, byte sizes, per-chunk
SHA-256 digests, and one domain-separated aggregate digest. Receipts must bind
those declarations to one fresh generation. The result either carries an
immutable conditional switch plan or reports a pointer conflict while keeping
the observed generation authoritative.

## When to Use

Use this pattern after another layer has staged a small, finite generation and
collected one trustworthy receipt for every declared chunk. Ordinals must start
at zero, IDs must be stable conservative ASCII tokens, and all producers must
use the same framing format for the aggregate digest.

The supplied current pointer is only an observation used to decide whether a
plan can be formed. An execution layer must still apply the expected-pointer
condition at the storage boundary because the pointer can change after this
pure function returns.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import StrEnum
from hashlib import sha256

_MAX_CHUNKS = 1_024
_MAX_CHUNK_BYTES = 1 << 30
_MAX_TOTAL_BYTES = 1 << 40
_GENERATION_ID = re.compile(r"[a-z0-9][a-z0-9._:-]{0,63}", re.ASCII)
_LOWER_SHA256 = re.compile(r"[0-9a-f]{64}", re.ASCII)
_AGGREGATE_DOMAIN = b"bounded-chunk-manifest-v1\x00"


@dataclass(frozen=True, slots=True)
class ManifestChunk:
    ordinal: int
    byte_count: int
    sha256_hex: str


@dataclass(frozen=True, slots=True)
class ExpectedChunkManifest:
    chunk_count: int
    total_bytes: int
    aggregate_sha256_hex: str
    chunks: tuple[ManifestChunk, ...]


@dataclass(frozen=True, slots=True)
class ChunkReceipt:
    generation_id: str
    ordinal: int
    byte_count: int
    sha256_hex: str


@dataclass(frozen=True, slots=True)
class ConditionalVersionSwitchPlan:
    expected_current_pointer: str
    replacement_generation_id: str
    chunk_count: int
    total_bytes: int
    aggregate_sha256_hex: str


class ManifestSwitchStatus(StrEnum):
    PLAN_READY = "plan-ready"
    CONFLICT = "conflict"


@dataclass(frozen=True, slots=True)
class ManifestSwitchDecision:
    status: ManifestSwitchStatus
    authoritative_generation_id: str
    plan: ConditionalVersionSwitchPlan | None
    orphaned_generation_id: str | None


def _validated_integer(
    value: object,
    *,
    field: str,
    minimum: int,
    maximum: int,
) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer")
    if not minimum <= value <= maximum:
        raise ValueError(f"{field} is outside the supported range")
    return value


def _validated_generation_id(value: object, *, field: str) -> str:
    if type(value) is not str or _GENERATION_ID.fullmatch(value) is None:
        raise ValueError(f"{field} must be a conservative 1-to-64-byte ID")
    return value


def _validated_digest(value: object, *, field: str) -> str:
    if type(value) is not str or _LOWER_SHA256.fullmatch(value) is None:
        raise ValueError(f"{field} must be a lowercase SHA-256 digest")
    return value


def _framed_aggregate_digest(chunks: tuple[ManifestChunk, ...]) -> str:
    digest = sha256()
    digest.update(_AGGREGATE_DOMAIN)
    digest.update(len(chunks).to_bytes(4, "big"))
    for chunk in chunks:
        digest.update(chunk.ordinal.to_bytes(4, "big"))
        digest.update(chunk.byte_count.to_bytes(8, "big"))
        digest.update(bytes.fromhex(chunk.sha256_hex))
    return digest.hexdigest()


def _validated_manifest(value: object) -> ExpectedChunkManifest:
    if type(value) is not ExpectedChunkManifest:
        raise TypeError("expected_manifest must be an exact ExpectedChunkManifest")
    chunk_count = _validated_integer(
        value.chunk_count,
        field="expected_manifest.chunk_count",
        minimum=1,
        maximum=_MAX_CHUNKS,
    )
    total_bytes = _validated_integer(
        value.total_bytes,
        field="expected_manifest.total_bytes",
        minimum=1,
        maximum=_MAX_TOTAL_BYTES,
    )
    aggregate = _validated_digest(
        value.aggregate_sha256_hex,
        field="expected_manifest.aggregate_sha256_hex",
    )
    if type(value.chunks) is not tuple:
        raise TypeError("expected_manifest.chunks must be an exact tuple")
    if len(value.chunks) != chunk_count:
        raise ValueError("expected_manifest.chunk_count does not match its chunks")

    chunks: list[ManifestChunk] = []
    for index, raw_chunk in enumerate(value.chunks):
        field = f"expected_manifest.chunks[{index}]"
        if type(raw_chunk) is not ManifestChunk:
            raise TypeError(f"{field} must be an exact ManifestChunk")
        chunks.append(
            ManifestChunk(
                ordinal=_validated_integer(
                    raw_chunk.ordinal,
                    field=f"{field}.ordinal",
                    minimum=0,
                    maximum=_MAX_CHUNKS - 1,
                ),
                byte_count=_validated_integer(
                    raw_chunk.byte_count,
                    field=f"{field}.byte_count",
                    minimum=1,
                    maximum=_MAX_CHUNK_BYTES,
                ),
                sha256_hex=_validated_digest(
                    raw_chunk.sha256_hex,
                    field=f"{field}.sha256_hex",
                ),
            )
        )

    ordinals = tuple(chunk.ordinal for chunk in chunks)
    if len(set(ordinals)) != len(ordinals):
        raise ValueError("expected manifest chunk ordinals must be unique")
    if ordinals != tuple(range(chunk_count)):
        raise ValueError("expected manifest chunks must be declaration-ordered and contiguous")
    if sum(chunk.byte_count for chunk in chunks) != total_bytes:
        raise ValueError("expected_manifest.total_bytes does not match its chunks")

    frozen_chunks = tuple(chunks)
    if _framed_aggregate_digest(frozen_chunks) != aggregate:
        raise ValueError("expected manifest aggregate digest does not match its chunks")
    return ExpectedChunkManifest(
        chunk_count=chunk_count,
        total_bytes=total_bytes,
        aggregate_sha256_hex=aggregate,
        chunks=frozen_chunks,
    )


def _validated_receipts(
    value: object,
    *,
    generation_id: str,
    manifest: ExpectedChunkManifest,
) -> tuple[ChunkReceipt, ...]:
    if type(value) is not tuple:
        raise TypeError("receipts must be an exact tuple")
    if not 1 <= len(value) <= _MAX_CHUNKS:
        raise ValueError(f"receipts must contain between 1 and {_MAX_CHUNKS} records")
    if len(value) != manifest.chunk_count:
        raise ValueError("receipt count does not match the expected manifest")

    receipts: list[ChunkReceipt] = []
    for index, raw_receipt in enumerate(value):
        field = f"receipts[{index}]"
        if type(raw_receipt) is not ChunkReceipt:
            raise TypeError(f"{field} must be an exact ChunkReceipt")
        receipts.append(
            ChunkReceipt(
                generation_id=_validated_generation_id(
                    raw_receipt.generation_id,
                    field=f"{field}.generation_id",
                ),
                ordinal=_validated_integer(
                    raw_receipt.ordinal,
                    field=f"{field}.ordinal",
                    minimum=0,
                    maximum=_MAX_CHUNKS - 1,
                ),
                byte_count=_validated_integer(
                    raw_receipt.byte_count,
                    field=f"{field}.byte_count",
                    minimum=1,
                    maximum=_MAX_CHUNK_BYTES,
                ),
                sha256_hex=_validated_digest(
                    raw_receipt.sha256_hex,
                    field=f"{field}.sha256_hex",
                ),
            )
        )

    ordinals = tuple(receipt.ordinal for receipt in receipts)
    if len(set(ordinals)) != len(ordinals):
        raise ValueError("receipt ordinals must be unique")
    if ordinals != tuple(range(manifest.chunk_count)):
        raise ValueError("receipts must be declaration-ordered and contiguous")

    receipt_chunks: list[ManifestChunk] = []
    for index, (receipt, expected) in enumerate(zip(receipts, manifest.chunks, strict=True)):
        if receipt.generation_id != generation_id:
            raise ValueError(f"receipts[{index}] belongs to another generation")
        if receipt.byte_count != expected.byte_count:
            raise ValueError(f"receipts[{index}] byte count does not match the manifest")
        if receipt.sha256_hex != expected.sha256_hex:
            raise ValueError(f"receipts[{index}] digest does not match the manifest")
        receipt_chunks.append(
            ManifestChunk(receipt.ordinal, receipt.byte_count, receipt.sha256_hex)
        )

    if sum(chunk.byte_count for chunk in receipt_chunks) != manifest.total_bytes:
        raise ValueError("receipt bytes do not match the expected total")
    if _framed_aggregate_digest(tuple(receipt_chunks)) != manifest.aggregate_sha256_hex:
        raise ValueError("receipt aggregate digest does not match the manifest")
    return tuple(receipts)


def validate_chunk_manifest_before_switch(
    expected_manifest: ExpectedChunkManifest,
    fresh_generation_id: str,
    receipts: tuple[ChunkReceipt, ...],
    *,
    expected_current_pointer: str,
    observed_current_pointer: str,
) -> ManifestSwitchDecision:
    """Validate staged receipts and form, but never execute, a switch plan."""
    manifest = _validated_manifest(expected_manifest)
    replacement = _validated_generation_id(
        fresh_generation_id,
        field="fresh_generation_id",
    )
    expected_current = _validated_generation_id(
        expected_current_pointer,
        field="expected_current_pointer",
    )
    observed_current = _validated_generation_id(
        observed_current_pointer,
        field="observed_current_pointer",
    )
    if replacement in (expected_current, observed_current):
        raise ValueError("fresh_generation_id must differ from every supplied pointer")

    _validated_receipts(
        receipts,
        generation_id=replacement,
        manifest=manifest,
    )
    if observed_current != expected_current:
        return ManifestSwitchDecision(
            status=ManifestSwitchStatus.CONFLICT,
            authoritative_generation_id=observed_current,
            plan=None,
            orphaned_generation_id=replacement,
        )

    return ManifestSwitchDecision(
        status=ManifestSwitchStatus.PLAN_READY,
        authoritative_generation_id=observed_current,
        plan=ConditionalVersionSwitchPlan(
            expected_current_pointer=expected_current,
            replacement_generation_id=replacement,
            chunk_count=manifest.chunk_count,
            total_bytes=manifest.total_bytes,
            aggregate_sha256_hex=manifest.aggregate_sha256_hex,
        ),
        orphaned_generation_id=None,
    )
```

## Example

```python
first_digest = sha256(b"alpha").hexdigest()
second_digest = sha256(b"beta").hexdigest()
chunks = (
    ManifestChunk(0, 5, first_digest),
    ManifestChunk(1, 4, second_digest),
)
manifest = ExpectedChunkManifest(
    chunk_count=2,
    total_bytes=9,
    aggregate_sha256_hex=_framed_aggregate_digest(chunks),
    chunks=chunks,
)
receipts = (
    ChunkReceipt("release-8", 0, 5, first_digest),
    ChunkReceipt("release-8", 1, 4, second_digest),
)

ready = validate_chunk_manifest_before_switch(
    manifest,
    "release-8",
    receipts,
    expected_current_pointer="release-7",
    observed_current_pointer="release-7",
)
conflict = validate_chunk_manifest_before_switch(
    manifest,
    "release-8",
    receipts,
    expected_current_pointer="release-7",
    observed_current_pointer="release-7-other",
)

assert ready == ManifestSwitchDecision(
    status=ManifestSwitchStatus.PLAN_READY,
    authoritative_generation_id="release-7",
    plan=ConditionalVersionSwitchPlan(
        expected_current_pointer="release-7",
        replacement_generation_id="release-8",
        chunk_count=2,
        total_bytes=9,
        aggregate_sha256_hex=manifest.aggregate_sha256_hex,
    ),
    orphaned_generation_id=None,
)
assert (conflict.status, conflict.authoritative_generation_id, conflict.plan) == (
    ManifestSwitchStatus.CONFLICT,
    "release-7-other",
    None,
)
assert conflict.orphaned_generation_id == "release-8"
```

## Trade-offs and Limitations

Validation is linear in at most 1,024 declarations and receipts. The aggregate
hash frames the count, ordinal, byte count, and raw per-chunk digest, so field
boundaries are unambiguous. It authenticates nothing: correctness still relies
on trustworthy receipts and immutable staged bytes whose digests were computed
from the intended contents. The caller must establish global generation-ID
freshness; this local check can only prove that the replacement differs from
the supplied pointers.

A plan-ready result is not a completed switch. The observed pointer remains
authoritative until an external conditional operation succeeds, and that
operation can still conflict after this snapshot. A conflict is not success;
the existing pointer remains authoritative and the staged generation may be
orphaned. This helper performs no upload, retry, timestamping, cleanup,
compare-and-swap execution, or I/O, and makes no atomicity, durability, or
exactly-once claim.

## Related Snippets

<!-- catalog:related:start -->
- [Validate a Bounded Stage-Verify-Pointer-Switch Log](validate-a-bounded-stage-verify-pointer-switch-log.md)
- [Fingerprint a Bounded Flat File Set with Framed SHA-256](fingerprint-a-bounded-flat-file-set-with-framed-sha-256.md)
- [Compare and Swap a Versioned SQLite Setting with One Conditional Update](compare-and-swap-a-versioned-sqlite-setting-with-one-conditional-update.md)
<!-- catalog:related:end -->
