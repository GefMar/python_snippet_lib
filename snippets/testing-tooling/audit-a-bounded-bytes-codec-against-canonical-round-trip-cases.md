---
title: "Audit a Bounded Bytes Codec Against Canonical Round-Trip Cases"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - compare-a-bounded-text-capture-against-a-golden-fixture.md
  - generate-a-bounded-one-edit-byte-mutation-corpus.md
  - ../configuration-serialization/encode-and-decode-canonical-bencode-under-structural-limits.md
---

# Audit a Bounded Bytes Codec Against Canonical Round-Trip Cases

## Idea and Problem

Audit a bytes codec against explicit canonical cases, detecting exceptions, unstable results, oversized outputs, non-canonical encodings, and round-trip mismatches.

Round-trip equality alone is insufficient: an encoder and decoder can agree on
the same incorrect or non-canonical representation. Each case therefore pairs
a value with independently reviewed canonical bytes. Repeating each operation
also exposes stateful behavior without retaining potentially sensitive output
in the diagnostic result.

## When to Use

Use this audit for small deterministic codecs whose input and decoded value are
bytes, such as compact framing, escaping, or canonical serialization layers.
Keep cases representative and independently derived from the format contract,
including empty, boundary, and structurally distinct values.

Use a dedicated conformance suite when aliases must be rejected by the decoder,
streams are involved, or a formal specification supplies a larger corpus. Add
property tests and fuzzing separately; neither can replace reviewed canonical
examples.

## Implementation

```python
from collections.abc import Callable
from dataclasses import dataclass
from enum import StrEnum


_MAX_CODEC_CASES = 32
_MAX_CODEC_VALUE_BYTES = 16 * 1_024
_MAX_CODEC_ENCODING_BYTES = 64 * 1_024
_MAX_CODEC_DECLARED_BYTES = 256 * 1_024


@dataclass(frozen=True, slots=True)
class BytesCodecCase:
    value: bytes
    canonical_encoding: bytes


class BytesCodecViolationKind(StrEnum):
    ENCODE_RAISED = "encode-raised"
    ENCODE_NON_BYTES = "encode-non-bytes"
    ENCODE_TOO_LARGE = "encode-too-large"
    ENCODE_UNSTABLE = "encode-unstable"
    ENCODE_NON_CANONICAL = "encode-non-canonical"
    DECODE_RAISED = "decode-raised"
    DECODE_NON_BYTES = "decode-non-bytes"
    DECODE_TOO_LARGE = "decode-too-large"
    DECODE_UNSTABLE = "decode-unstable"
    ROUND_TRIP_MISMATCH = "round-trip-mismatch"


@dataclass(frozen=True, slots=True)
class BytesCodecViolation:
    kind: BytesCodecViolationKind
    case_index: int
    call_index: int | None = None


@dataclass(frozen=True, slots=True)
class BytesCodecAudit:
    violation: BytesCodecViolation | None

    @property
    def valid(self) -> bool:
        return self.violation is None


def _validate_codec_cases(cases: object) -> tuple[BytesCodecCase, ...]:
    if type(cases) is not tuple:
        raise TypeError("cases must be an exact tuple")
    if not 1 <= len(cases) <= _MAX_CODEC_CASES:
        raise ValueError("cases must contain between 1 and 32 entries")

    values: set[bytes] = set()
    encodings: set[bytes] = set()
    declared_bytes = 0
    for case in cases:
        if type(case) is not BytesCodecCase:
            raise TypeError("every case must be an exact BytesCodecCase")
        if type(case.value) is not bytes or type(case.canonical_encoding) is not bytes:
            raise TypeError("case values and encodings must be exact bytes")
        if len(case.value) > _MAX_CODEC_VALUE_BYTES:
            raise ValueError("a case value is too large")
        if len(case.canonical_encoding) > _MAX_CODEC_ENCODING_BYTES:
            raise ValueError("a canonical encoding is too large")
        if case.value in values:
            raise ValueError("case values must be unique")
        if case.canonical_encoding in encodings:
            raise ValueError("canonical encodings must be unique")
        values.add(case.value)
        encodings.add(case.canonical_encoding)
        declared_bytes += len(case.value) + len(case.canonical_encoding)

    if declared_bytes > _MAX_CODEC_DECLARED_BYTES:
        raise ValueError("the aggregate declared case data is too large")
    return cases


def _observe_codec_call(
    operation: Callable[[bytes], bytes],
    argument: bytes,
    *,
    case_index: int,
    call_index: int,
    maximum_bytes: int,
    raised_kind: BytesCodecViolationKind,
    non_bytes_kind: BytesCodecViolationKind,
    too_large_kind: BytesCodecViolationKind,
) -> bytes | BytesCodecViolation:
    try:
        result = operation(argument)
    except Exception:
        return BytesCodecViolation(
            raised_kind,
            case_index,
            call_index,
        )
    if type(result) is not bytes:
        return BytesCodecViolation(
            non_bytes_kind,
            case_index,
            call_index,
        )
    if len(result) > maximum_bytes:
        return BytesCodecViolation(too_large_kind, case_index, call_index)
    return result


def audit_bytes_codec(
    cases: tuple[BytesCodecCase, ...],
    encode: Callable[[bytes], bytes],
    decode: Callable[[bytes], bytes],
) -> BytesCodecAudit:
    validated_cases = _validate_codec_cases(cases)
    if not callable(encode) or not callable(decode):
        raise TypeError("encode and decode must be callable")

    for case_index, case in enumerate(validated_cases):
        encoded: list[bytes] = []
        for call_index in (1, 2):
            observation = _observe_codec_call(
                encode,
                case.value,
                case_index=case_index,
                call_index=call_index,
                maximum_bytes=_MAX_CODEC_ENCODING_BYTES,
                raised_kind=BytesCodecViolationKind.ENCODE_RAISED,
                non_bytes_kind=BytesCodecViolationKind.ENCODE_NON_BYTES,
                too_large_kind=BytesCodecViolationKind.ENCODE_TOO_LARGE,
            )
            if type(observation) is BytesCodecViolation:
                return BytesCodecAudit(observation)
            encoded.append(observation)

        if encoded[0] != encoded[1]:
            return BytesCodecAudit(
                BytesCodecViolation(
                    BytesCodecViolationKind.ENCODE_UNSTABLE,
                    case_index,
                )
            )
        if encoded[0] != case.canonical_encoding:
            return BytesCodecAudit(
                BytesCodecViolation(
                    BytesCodecViolationKind.ENCODE_NON_CANONICAL,
                    case_index,
                )
            )

        decoded: list[bytes] = []
        for call_index in (1, 2):
            observation = _observe_codec_call(
                decode,
                case.canonical_encoding,
                case_index=case_index,
                call_index=call_index,
                maximum_bytes=_MAX_CODEC_VALUE_BYTES,
                raised_kind=BytesCodecViolationKind.DECODE_RAISED,
                non_bytes_kind=BytesCodecViolationKind.DECODE_NON_BYTES,
                too_large_kind=BytesCodecViolationKind.DECODE_TOO_LARGE,
            )
            if type(observation) is BytesCodecViolation:
                return BytesCodecAudit(observation)
            decoded.append(observation)

        if decoded[0] != decoded[1]:
            return BytesCodecAudit(
                BytesCodecViolation(
                    BytesCodecViolationKind.DECODE_UNSTABLE,
                    case_index,
                )
            )
        if decoded[0] != case.value:
            return BytesCodecAudit(
                BytesCodecViolation(
                    BytesCodecViolationKind.ROUND_TRIP_MISMATCH,
                    case_index,
                )
            )

    return BytesCodecAudit(violation=None)
```

## Example

```python
from itertools import product


def assert_raises(error_type, operation):
    try:
        operation()
    except error_type:
        return
    raise AssertionError(f"{error_type.__name__} was not raised")


events = []


def encode_frame(value):
    events.append(("encode", value))
    return len(value).to_bytes(2, "big") + value


def decode_frame(encoded):
    events.append(("decode", encoded))
    if len(encoded) < 2:
        raise ValueError("truncated frame")
    declared_size = int.from_bytes(encoded[:2], "big")
    value = encoded[2:]
    if len(value) != declared_size:
        raise ValueError("incorrect frame size")
    return value


# The expected frames use a separate construction over a small exhaustive
# alphabet rather than calling the encoder under audit.
values = (
    *(bytes(items) for size in range(5) for items in product((0, 255), repeat=size)),
    b"\x02",
)
cases = tuple(BytesCodecCase(value, bytes((0, len(value))) + value) for value in values)
assert len(cases) == 32
assert audit_bytes_codec(cases, encode_frame, decode_frame).valid
assert len(events) == 128
assert all(
    [operation for operation, _ in events[offset : offset + 4]]
    == ["encode", "encode", "decode", "decode"]
    for offset in range(0, len(events), 4)
)

maximum_value = b"v" * (16 * 1_024)
maximum_value_case = BytesCodecCase(
    maximum_value,
    b"\x40\x00" + maximum_value,
)
assert audit_bytes_codec((maximum_value_case,), encode_frame, decode_frame).valid

maximum_encoding = b"x" * (64 * 1_024)
assert audit_bytes_codec(
    (BytesCodecCase(b"value", maximum_encoding),),
    lambda value: maximum_encoding,
    lambda encoded: b"value",
).valid

aggregate_values = tuple(bytes((index,)) for index in range(1, 5))
aggregate_encodings = (
    b"a" * 65_536,
    b"b" * 65_535,
    b"c" * 65_534,
    b"d" * 65_535,
)
aggregate_cases = tuple(
    BytesCodecCase(value, encoding)
    for value, encoding in zip(aggregate_values, aggregate_encodings, strict=True)
)
encoding_by_value = dict(zip(aggregate_values, aggregate_encodings, strict=True))
value_by_encoding = dict(zip(aggregate_encodings, aggregate_values, strict=True))
assert audit_bytes_codec(
    aggregate_cases,
    encoding_by_value.__getitem__,
    value_by_encoding.__getitem__,
).valid

unexpected_callbacks = []
over_aggregate_encodings = (
    b"a" * 65_536,
    b"b" * 65_536,
    b"c" * 65_534,
    b"d" * 65_535,
)
over_aggregate_cases = tuple(
    BytesCodecCase(value, encoding)
    for value, encoding in zip(aggregate_values, over_aggregate_encodings, strict=True)
)


def unexpected_callback(value):
    unexpected_callbacks.append(value)
    return b""


assert_raises(
    ValueError,
    lambda: audit_bytes_codec(over_aggregate_cases, unexpected_callback, unexpected_callback),
)
assert unexpected_callbacks == []


def violation_for(encode, decode=lambda encoded: b"value"):
    return audit_bytes_codec((BytesCodecCase(b"value", b"canonical"),), encode, decode).violation


def raise_lookup_error(value):
    raise LookupError


assert violation_for(raise_lookup_error).kind is BytesCodecViolationKind.ENCODE_RAISED
assert violation_for(raise_lookup_error).call_index == 1
assert violation_for(lambda value: bytearray()).kind is BytesCodecViolationKind.ENCODE_NON_BYTES
assert violation_for(lambda value: b"x" * 65_537).kind is BytesCodecViolationKind.ENCODE_TOO_LARGE

encode_outputs = iter((b"canonical", b"different"))
assert (
    violation_for(lambda value: next(encode_outputs)).kind
    is BytesCodecViolationKind.ENCODE_UNSTABLE
)

stopped_at_noncanonical = []


def noncanonical_encode(value):
    stopped_at_noncanonical.append("encode")
    return b"different"


def should_not_decode(encoded):
    stopped_at_noncanonical.append("decode")
    return b"value"


assert (
    violation_for(noncanonical_encode, should_not_decode).kind
    is BytesCodecViolationKind.ENCODE_NON_CANONICAL
)
assert stopped_at_noncanonical == ["encode", "encode"]


def canonical_encode(value):
    return b"canonical"


assert (
    violation_for(canonical_encode, raise_lookup_error).kind
    is BytesCodecViolationKind.DECODE_RAISED
)
assert (
    violation_for(canonical_encode, lambda encoded: bytearray()).kind
    is BytesCodecViolationKind.DECODE_NON_BYTES
)
assert (
    violation_for(canonical_encode, lambda encoded: b"x" * 16_385).kind
    is BytesCodecViolationKind.DECODE_TOO_LARGE
)

decode_outputs = iter((b"value", b"different"))
assert (
    violation_for(canonical_encode, lambda encoded: next(decode_outputs)).kind
    is BytesCodecViolationKind.DECODE_UNSTABLE
)
assert (
    violation_for(canonical_encode, lambda encoded: b"different").kind
    is BytesCodecViolationKind.ROUND_TRIP_MISMATCH
)

assert_raises(
    ValueError,
    lambda: audit_bytes_codec(
        tuple(BytesCodecCase(bytes((index,)), bytes((index,))) for index in range(33)),
        encode_frame,
        decode_frame,
    ),
)
assert_raises(
    ValueError,
    lambda: audit_bytes_codec(
        (BytesCodecCase(b"x", b"a"), BytesCodecCase(b"x", b"b")),
        encode_frame,
        decode_frame,
    ),
)
assert_raises(
    ValueError,
    lambda: audit_bytes_codec(
        (BytesCodecCase(b"x", b"a"), BytesCodecCase(b"y", b"a")),
        encode_frame,
        decode_frame,
    ),
)
assert_raises(
    ValueError,
    lambda: audit_bytes_codec((BytesCodecCase(b"x" * 16_385, b"a"),), encode_frame, decode_frame),
)
assert_raises(
    ValueError,
    lambda: audit_bytes_codec((BytesCodecCase(b"x", b"a" * 65_537),), encode_frame, decode_frame),
)


class StopAudit(BaseException):
    pass


def stop_audit(value):
    raise StopAudit


try:
    audit_bytes_codec((BytesCodecCase(b"value", b"canonical"),), stop_audit, decode_frame)
except StopAudit:
    pass
else:
    raise AssertionError("BaseException must propagate")

assert audit_bytes_codec(
    (BytesCodecCase(b"", b"\x00\x00"),),
    encode_frame,
    decode_frame,
).valid
```

## Trade-offs and Limitations

A successful audit performs exactly four callback calls per case, for at most
128 calls. It stops at the first violation and bounds retained evidence, but an
output size can be checked only after a callback returns. Trusted callbacks can
still block, allocate excessively, perform I/O, or mutate external state;
ordinary `Exception` instances become violations while `BaseException`
instances propagate.

The audit proves behavior only for the declared corpus. It does not check
whether a decoder accepts non-canonical aliases, nor does it assess streaming,
lossy formats, timing behavior, security properties, or malformed-input
coverage. Canonical cases must come from an oracle independent of the codec
implementation.

## Related Snippets

<!-- catalog:related:start -->
- [Compare a Bounded Text Capture Against a Golden Fixture](compare-a-bounded-text-capture-against-a-golden-fixture.md)
- [Generate a Bounded One-Edit Byte Mutation Corpus](generate-a-bounded-one-edit-byte-mutation-corpus.md)
- [Encode and Decode Canonical Bencode Under Structural Limits](../configuration-serialization/encode-and-decode-canonical-bencode-under-structural-limits.md)
<!-- catalog:related:end -->
