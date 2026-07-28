---
title: "Plan Recovery Across Object and Metadata Publication States"
snippet_type: algorithm
use_cases:
  - lifecycle-management
  - persistence
  - retry-recovery
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../storage-databases/validate-a-bounded-chunk-manifest-before-a-conditional-version-switch.md
  - ../storage-databases/store-bytes-by-their-content-digest.md
  - plan-a-versioned-transition-for-the-current-workflow-attempt.md
---

# Plan Recovery Across Object and Metadata Publication States

## Idea and Problem

Choose one safe recovery action from explicit object and metadata observations for a fixed publication request.

The request binds a stable operation ID to one immutable object key, digest,
and byte size. Each observation is explicitly unknown, absent, or present;
present observations carry the complete identity needed for exact comparison.
The planner never proposes metadata publication until a matching durable object
has been observed.

## When to Use

Use this algorithm when object bytes and their published metadata live behind
separate interfaces and an interrupted executor must decide only its next
step. An object `PRESENT` observation must mean that an authoritative read has
confirmed durable storage, not merely that an upload request returned.

The object key must be immutable once populated. Both object publication and
metadata publication must support atomic create-if-absent operations. After a
precondition miss or any failed, timed-out, or otherwise ambiguous executor
call, observe both sides again instead of repeating the planned write blindly.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import StrEnum

_MAX_OPERATION_ID_BYTES = 64
_MAX_OBJECT_KEY_BYTES = 128
_MAX_OBJECT_BYTES = (1 << 63) - 1
_TOKEN = re.compile(r"[a-z0-9][a-z0-9._:-]*", re.ASCII)
_LOWER_SHA256 = re.compile(r"[0-9a-f]{64}", re.ASCII)


class ObservationState(StrEnum):
    UNKNOWN = "unknown"
    ABSENT = "absent"
    PRESENT = "present"


class RecoveryAction(StrEnum):
    REOBSERVE = "reobserve"
    CONDITIONALLY_CREATE_OBJECT = "conditionally-create-object"
    CONDITIONALLY_PUBLISH_METADATA = "conditionally-publish-metadata"
    COMPLETE = "complete"
    CONFLICT = "conflict"


class RecoveryReason(StrEnum):
    OBSERVATION_UNKNOWN = "observation-unknown"
    OBJECT_AND_METADATA_ABSENT = "object-and-metadata-absent"
    MATCHING_OBJECT_METADATA_ABSENT = "matching-object-metadata-absent"
    OBJECT_AND_METADATA_MATCH = "object-and-metadata-match"
    OBJECT_IDENTITY_MISMATCH = "object-identity-mismatch"
    METADATA_IDENTITY_MISMATCH = "metadata-identity-mismatch"
    METADATA_WITHOUT_OBJECT = "metadata-without-object"


@dataclass(frozen=True, slots=True)
class PublicationRequest:
    operation_id: str
    object_key: str
    content_digest: str
    byte_size: int


@dataclass(frozen=True, slots=True)
class ObjectIdentity:
    object_key: str
    content_digest: str
    byte_size: int


@dataclass(frozen=True, slots=True)
class PublishedMetadataIdentity:
    operation_id: str
    object_key: str
    content_digest: str
    byte_size: int


@dataclass(frozen=True, slots=True)
class ObjectObservation:
    state: ObservationState
    identity: ObjectIdentity | None = None


@dataclass(frozen=True, slots=True)
class MetadataObservation:
    state: ObservationState
    identity: PublishedMetadataIdentity | None = None


@dataclass(frozen=True, slots=True)
class PublicationRecoveryPlan:
    action: RecoveryAction
    reason: RecoveryReason
    request: PublicationRequest


def _validated_token(
    value: object,
    *,
    field: str,
    byte_limit: int,
) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not 1 <= len(value) <= byte_limit or _TOKEN.fullmatch(value) is None:
        raise ValueError(f"{field} must be a bounded conservative ASCII token")
    return value


def _validated_digest(value: object, *, field: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{field} must be an exact string")
    if _LOWER_SHA256.fullmatch(value) is None:
        raise ValueError(f"{field} must be a lowercase SHA-256 digest")
    return value


def _validated_size(value: object, *, field: str) -> int:
    if type(value) is not int:
        raise TypeError(f"{field} must be an exact integer")
    if not 0 <= value <= _MAX_OBJECT_BYTES:
        raise ValueError(f"{field} is outside the supported range")
    return value


def _validated_request(value: object) -> PublicationRequest:
    if type(value) is not PublicationRequest:
        raise TypeError("request must be an exact PublicationRequest")
    return PublicationRequest(
        operation_id=_validated_token(
            value.operation_id,
            field="request.operation_id",
            byte_limit=_MAX_OPERATION_ID_BYTES,
        ),
        object_key=_validated_token(
            value.object_key,
            field="request.object_key",
            byte_limit=_MAX_OBJECT_KEY_BYTES,
        ),
        content_digest=_validated_digest(
            value.content_digest,
            field="request.content_digest",
        ),
        byte_size=_validated_size(value.byte_size, field="request.byte_size"),
    )


def _validated_object_identity(value: object, *, field: str) -> ObjectIdentity:
    if type(value) is not ObjectIdentity:
        raise TypeError(f"{field} must be an exact ObjectIdentity")
    return ObjectIdentity(
        object_key=_validated_token(
            value.object_key,
            field=f"{field}.object_key",
            byte_limit=_MAX_OBJECT_KEY_BYTES,
        ),
        content_digest=_validated_digest(
            value.content_digest,
            field=f"{field}.content_digest",
        ),
        byte_size=_validated_size(value.byte_size, field=f"{field}.byte_size"),
    )


def _validated_metadata_identity(
    value: object,
    *,
    field: str,
) -> PublishedMetadataIdentity:
    if type(value) is not PublishedMetadataIdentity:
        raise TypeError(f"{field} must be an exact PublishedMetadataIdentity")
    return PublishedMetadataIdentity(
        operation_id=_validated_token(
            value.operation_id,
            field=f"{field}.operation_id",
            byte_limit=_MAX_OPERATION_ID_BYTES,
        ),
        object_key=_validated_token(
            value.object_key,
            field=f"{field}.object_key",
            byte_limit=_MAX_OBJECT_KEY_BYTES,
        ),
        content_digest=_validated_digest(
            value.content_digest,
            field=f"{field}.content_digest",
        ),
        byte_size=_validated_size(value.byte_size, field=f"{field}.byte_size"),
    )


def _validated_object_observation(value: object) -> ObjectObservation:
    if type(value) is not ObjectObservation:
        raise TypeError("object_observation must be an exact ObjectObservation")
    if type(value.state) is not ObservationState:
        raise TypeError("object_observation.state must be an ObservationState")
    if value.state is ObservationState.PRESENT:
        identity = _validated_object_identity(
            value.identity,
            field="object_observation.identity",
        )
        return ObjectObservation(value.state, identity)
    if value.identity is not None:
        raise ValueError("a non-present object observation cannot carry identity")
    return ObjectObservation(value.state)


def _validated_metadata_observation(value: object) -> MetadataObservation:
    if type(value) is not MetadataObservation:
        raise TypeError("metadata_observation must be an exact MetadataObservation")
    if type(value.state) is not ObservationState:
        raise TypeError("metadata_observation.state must be an ObservationState")
    if value.state is ObservationState.PRESENT:
        identity = _validated_metadata_identity(
            value.identity,
            field="metadata_observation.identity",
        )
        return MetadataObservation(value.state, identity)
    if value.identity is not None:
        raise ValueError("a non-present metadata observation cannot carry identity")
    return MetadataObservation(value.state)


def plan_publication_recovery(
    request: PublicationRequest,
    *,
    object_observation: ObjectObservation,
    metadata_observation: MetadataObservation,
) -> PublicationRecoveryPlan:
    publication = _validated_request(request)
    observed_object = _validated_object_observation(object_observation)
    observed_metadata = _validated_metadata_observation(metadata_observation)

    if (
        observed_object.state is ObservationState.UNKNOWN
        or observed_metadata.state is ObservationState.UNKNOWN
    ):
        return PublicationRecoveryPlan(
            RecoveryAction.REOBSERVE,
            RecoveryReason.OBSERVATION_UNKNOWN,
            publication,
        )

    expected_object = ObjectIdentity(
        publication.object_key,
        publication.content_digest,
        publication.byte_size,
    )
    expected_metadata = PublishedMetadataIdentity(
        publication.operation_id,
        publication.object_key,
        publication.content_digest,
        publication.byte_size,
    )

    if (
        observed_metadata.state is ObservationState.PRESENT
        and observed_metadata.identity != expected_metadata
    ):
        return PublicationRecoveryPlan(
            RecoveryAction.CONFLICT,
            RecoveryReason.METADATA_IDENTITY_MISMATCH,
            publication,
        )
    if (
        observed_object.state is ObservationState.PRESENT
        and observed_object.identity != expected_object
    ):
        return PublicationRecoveryPlan(
            RecoveryAction.CONFLICT,
            RecoveryReason.OBJECT_IDENTITY_MISMATCH,
            publication,
        )

    if observed_object.state is ObservationState.ABSENT:
        if observed_metadata.state is ObservationState.PRESENT:
            return PublicationRecoveryPlan(
                RecoveryAction.CONFLICT,
                RecoveryReason.METADATA_WITHOUT_OBJECT,
                publication,
            )
        return PublicationRecoveryPlan(
            RecoveryAction.CONDITIONALLY_CREATE_OBJECT,
            RecoveryReason.OBJECT_AND_METADATA_ABSENT,
            publication,
        )

    if observed_metadata.state is ObservationState.ABSENT:
        return PublicationRecoveryPlan(
            RecoveryAction.CONDITIONALLY_PUBLISH_METADATA,
            RecoveryReason.MATCHING_OBJECT_METADATA_ABSENT,
            publication,
        )
    return PublicationRecoveryPlan(
        RecoveryAction.COMPLETE,
        RecoveryReason.OBJECT_AND_METADATA_MATCH,
        publication,
    )
```

## Example

```python
request = PublicationRequest(
    operation_id="publication-17",
    object_key="artifact-blue",
    content_digest="4" * 64,
    byte_size=640,
)
object_identity = ObjectIdentity("artifact-blue", "4" * 64, 640)
metadata_identity = PublishedMetadataIdentity(
    "publication-17",
    "artifact-blue",
    "4" * 64,
    640,
)

create_object = plan_publication_recovery(
    request,
    object_observation=ObjectObservation(ObservationState.ABSENT),
    metadata_observation=MetadataObservation(ObservationState.ABSENT),
)
publish = plan_publication_recovery(
    request,
    object_observation=ObjectObservation(
        ObservationState.PRESENT,
        object_identity,
    ),
    metadata_observation=MetadataObservation(ObservationState.ABSENT),
)
complete = plan_publication_recovery(
    request,
    object_observation=ObjectObservation(
        ObservationState.PRESENT,
        object_identity,
    ),
    metadata_observation=MetadataObservation(
        ObservationState.PRESENT,
        metadata_identity,
    ),
)
reobserve = plan_publication_recovery(
    request,
    object_observation=ObjectObservation(ObservationState.UNKNOWN),
    metadata_observation=MetadataObservation(ObservationState.ABSENT),
)
conflict = plan_publication_recovery(
    request,
    object_observation=ObjectObservation(
        ObservationState.PRESENT,
        object_identity,
    ),
    metadata_observation=MetadataObservation(
        ObservationState.PRESENT,
        PublishedMetadataIdentity(
            "publication-16",
            "artifact-blue",
            "4" * 64,
            640,
        ),
    ),
)

assert (
    create_object.action,
    publish.action,
    complete.action,
    reobserve.action,
    conflict.action,
    conflict.reason,
) == (
    RecoveryAction.CONDITIONALLY_CREATE_OBJECT,
    RecoveryAction.CONDITIONALLY_PUBLISH_METADATA,
    RecoveryAction.COMPLETE,
    RecoveryAction.REOBSERVE,
    RecoveryAction.CONFLICT,
    RecoveryReason.METADATA_IDENTITY_MISMATCH,
)
```

## Trade-offs and Limitations

Planning uses constant work and memory. Operation IDs and object keys have
explicit ASCII byte limits, digests have one exact shape, and byte sizes are
bounded nonnegative integers. Exact dataclass and enum checks reject malformed
observation shapes before any state combination is evaluated.

`PRESENT` is meaningful only when supplied by an authoritative observation of
durable state. Conditional object creation and conditional metadata
publication prevent overwriting unexpected values within their respective
stores, but they do not make the stores atomic together. A create-if-absent
precondition miss requires fresh observations rather than an overwrite. The
executor owns retention and any immediate recheck needed by its consistency
model.

The planner performs no I/O, upload, publication, cleanup, retry, random-ID
generation, clock access, or adapter selection. `COMPLETE` reports a matching
observation, not a new durability guarantee. Any ambiguous executor outcome
requires fresh observations before this one-step planner is called again.

## Related Snippets

<!-- catalog:related:start -->
- [Validate a Bounded Chunk Manifest Before a Conditional Version Switch](../storage-databases/validate-a-bounded-chunk-manifest-before-a-conditional-version-switch.md)
- [Store Bytes by Their Content Digest](../storage-databases/store-bytes-by-their-content-digest.md)
- [Plan a Versioned Transition for the Current Workflow Attempt](plan-a-versioned-transition-for-the-current-workflow-attempt.md)
<!-- catalog:related:end -->
