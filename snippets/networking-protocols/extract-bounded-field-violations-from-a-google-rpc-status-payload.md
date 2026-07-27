---
title: "Extract Bounded Field Violations from a google.rpc Status Payload"
snippet_type: integration
use_cases:
  - networking
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: googleapis-common-protos
    version: "1.75.0"
  - name: protobuf
    version: "7.35.1"
related:
  - parse-a-bounded-three-state-json-response-envelope.md
  - read-and-write-size-capped-varint-frames.md
  - ../data-processing/collect-expected-parse-failures-without-stopping-a-batch.md
---

# Extract Bounded Field Violations from a google.rpc Status Payload

## Idea and Problem

Decode field-level validation failures from one size-capped serialized Google RPC status value without depending on a live RPC object.

The parser recognizes at most one packed `google.rpc.BadRequest` detail and
returns its field violations in wire order. Every other detail is reported by
its bounded type URL and original position, so forward-compatible input is
visible without dynamic message lookup or accidental interpretation.

## When to Use

Use this integration after a trusted transport adapter has already extracted a
serialized status payload and the caller needs structured field problems. Keep
the returned values as protocol data; decide separately how much text is safe
to display or log.

Keep transport metadata extraction, retry policy, authentication, localization,
and user-interface rendering outside this parser. If an API uses a different
error-detail schema, decode that schema explicitly instead of guessing from a
human-readable status message.

## Implementation

```python
from dataclasses import dataclass

from google.protobuf.message import DecodeError
from google.rpc import code_pb2, error_details_pb2, status_pb2


_MAX_PAYLOAD_BYTES = 65_536
_MAX_DETAILS = 32
_MAX_PACKED_DETAIL_BYTES = 16_384
_MAX_VIOLATIONS = 128
_MAX_STATUS_MESSAGE_BYTES = 4_096
_MAX_TYPE_URL_BYTES = 512
_MAX_FIELD_BYTES = 512
_MAX_DESCRIPTION_BYTES = 4_096
_MAX_TOTAL_TEXT_BYTES = 65_536
_ERROR_CODES = frozenset(code_pb2.Code.values()) - {code_pb2.OK}


class StatusPayloadError(ValueError):
    pass


@dataclass(frozen=True, slots=True)
class BoundedFieldViolation:
    field: str
    description: str


@dataclass(frozen=True, slots=True)
class UnknownStatusDetail:
    detail_index: int
    type_url: str


@dataclass(frozen=True, slots=True)
class StatusFieldViolationReport:
    code: int
    violations: tuple[BoundedFieldViolation, ...]
    unknown_details: tuple[UnknownStatusDetail, ...]


@dataclass(slots=True)
class _TextBudget:
    used_bytes: int = 0


def _invalid() -> None:
    raise StatusPayloadError("invalid google.rpc status payload")


def _count_utf8_text(
    value: object,
    *,
    per_value_limit: int,
    budget: _TextBudget,
) -> str:
    if type(value) is not str:
        _invalid()
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError:
        _invalid()
    if len(encoded) > per_value_limit:
        _invalid()
    budget.used_bytes += len(encoded)
    if budget.used_bytes > _MAX_TOTAL_TEXT_BYTES:
        _invalid()
    return value


def _visible_text(
    value: object,
    *,
    per_value_limit: int,
    budget: _TextBudget,
) -> str:
    text = _count_utf8_text(
        value,
        per_value_limit=per_value_limit,
        budget=budget,
    )
    if not text or text != text.strip() or not text.isprintable():
        _invalid()
    return text


def _type_url(value: object, *, budget: _TextBudget) -> str:
    text = _visible_text(
        value,
        per_value_limit=_MAX_TYPE_URL_BYTES,
        budget=budget,
    )
    try:
        text.encode("ascii")
    except UnicodeEncodeError:
        _invalid()
    if "/" not in text:
        _invalid()
    return text


def extract_bounded_field_violations(
    payload: bytes,
) -> StatusFieldViolationReport:
    if type(payload) is not bytes:
        raise TypeError("payload must be exact bytes")
    if not 1 <= len(payload) <= _MAX_PAYLOAD_BYTES:
        _invalid()

    try:
        status = status_pb2.Status.FromString(payload)
    except DecodeError:
        _invalid()
    if status.code not in _ERROR_CODES:
        _invalid()
    if len(status.details) > _MAX_DETAILS:
        _invalid()

    budget = _TextBudget()
    _count_utf8_text(
        status.message,
        per_value_limit=_MAX_STATUS_MESSAGE_BYTES,
        budget=budget,
    )

    violations: list[BoundedFieldViolation] = []
    unknown_details: list[UnknownStatusDetail] = []
    bad_request_seen = False

    for detail_index, packed in enumerate(status.details):
        if len(packed.value) > _MAX_PACKED_DETAIL_BYTES:
            _invalid()
        type_url = _type_url(packed.type_url, budget=budget)
        if not packed.Is(error_details_pb2.BadRequest.DESCRIPTOR):
            unknown_details.append(UnknownStatusDetail(detail_index, type_url))
            continue
        if bad_request_seen:
            _invalid()
        bad_request_seen = True

        bad_request = error_details_pb2.BadRequest()
        try:
            unpacked = packed.Unpack(bad_request)
        except DecodeError:
            _invalid()
        if not unpacked:
            _invalid()
        if len(bad_request.field_violations) > _MAX_VIOLATIONS:
            _invalid()

        for violation in bad_request.field_violations:
            field = _count_utf8_text(
                violation.field,
                per_value_limit=_MAX_FIELD_BYTES,
                budget=budget,
            )
            description = _count_utf8_text(
                violation.description,
                per_value_limit=_MAX_DESCRIPTION_BYTES,
                budget=budget,
            )
            violations.append(BoundedFieldViolation(field, description))

    return StatusFieldViolationReport(
        code=status.code,
        violations=tuple(violations),
        unknown_details=tuple(unknown_details),
    )
```

## Example

```python
from google.protobuf.any_pb2 import Any


bad_request = error_details_pb2.BadRequest()
bad_request.field_violations.add(
    field="display_name",
    description="must not be empty\nincluding whitespace-only input",
)
bad_request.field_violations.add(
    field="page_size",
    description="must be between 1 and 100",
)
packed_bad_request = Any()
packed_bad_request.Pack(bad_request)

unknown = Any(
    type_url="type.example.test/example.OtherDetail",
    value=b"\x08\x01",
)
payload = status_pb2.Status(
    code=code_pb2.INVALID_ARGUMENT,
    message="request validation failed",
    details=(unknown, packed_bad_request),
).SerializeToString()

report = extract_bounded_field_violations(payload)

duplicate_payload = status_pb2.Status(
    code=code_pb2.INVALID_ARGUMENT,
    details=(packed_bad_request, packed_bad_request),
).SerializeToString()
oversized_detail_payload = status_pb2.Status(
    code=code_pb2.INVALID_ARGUMENT,
    details=(
        Any(
            type_url="type.example.test/example.OversizedDetail",
            value=b"x" * (_MAX_PACKED_DETAIL_BYTES + 1),
        ),
    ),
).SerializeToString()
rejected = 0
for invalid_payload in (
    b"\x80",
    duplicate_payload,
    oversized_detail_payload,
    b"x" * (_MAX_PAYLOAD_BYTES + 1),
):
    try:
        extract_bounded_field_violations(invalid_payload)
    except StatusPayloadError:
        rejected += 1

try:
    extract_bounded_field_violations(bytearray(payload))
except TypeError:
    mutable_input_rejected = True
else:
    mutable_input_rejected = False

try:
    report.violations[0].field = "changed"
except (AttributeError, TypeError):
    immutable = True
else:
    immutable = False

assert (
    report,
    rejected,
    mutable_input_rejected,
    immutable,
) == (
    StatusFieldViolationReport(
        code=code_pb2.INVALID_ARGUMENT,
        violations=(
            BoundedFieldViolation(
                "display_name",
                "must not be empty\nincluding whitespace-only input",
            ),
            BoundedFieldViolation("page_size", "must be between 1 and 100"),
        ),
        unknown_details=(
            UnknownStatusDetail(
                0,
                "type.example.test/example.OtherDetail",
            ),
        ),
    ),
    4,
    True,
    True,
)
```

## Trade-offs and Limitations

Payload, detail-count, packed-detail, violation, per-field, and aggregate text
limits bound parsing and retained output. Unknown detail values remain inside
the bounded payload and must also satisfy the per-detail byte limit, but are
not decoded; the report records only their positions and type URLs. Protobuf
unknown fields are left to the runtime's normal forward-compatibility behavior.

The parser accepts only canonical non-OK `google.rpc.Code` values and at most
one `BadRequest` detail. It does not obtain status bytes from trailing metadata,
load message classes dynamically, interpret other detail schemas, render or
redact descriptions, perform an RPC, log content, or decide whether an
operation should be retried.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Three-State JSON Response Envelope](parse-a-bounded-three-state-json-response-envelope.md)
- [Read and Write Size-Capped Varint Frames](read-and-write-size-capped-varint-frames.md)
- [Collect Expected Parse Failures Without Stopping a Batch](../data-processing/collect-expected-parse-failures-without-stopping-a-batch.md)
<!-- catalog:related:end -->
