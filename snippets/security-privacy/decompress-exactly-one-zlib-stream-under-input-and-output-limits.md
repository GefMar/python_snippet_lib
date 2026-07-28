---
title: "Decompress Exactly One Zlib Stream Under Input and Output Limits"
snippet_type: recipe
use_cases:
  - resource-management
  - security
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - verify-a-bounded-byte-stream-before-returning-its-payload.md
  - ../configuration-serialization/build-a-deterministic-size-capped-ustar-archive-from-bytes.md
  - ../networking-protocols/release-a-pooled-response-connection-only-after-clean-eof.md
---

# Decompress Exactly One Zlib Stream Under Input and Output Limits

## Idea and Problem

Decompress one complete zlib-wrapped byte stream while bounding both admitted input and returned output.

The compressed byte limit is checked before constructing a decompressor. The
decompressor receives an output allowance of one byte beyond the caller's
limit, making an oversized result observable without allowing an unbounded
allocation. End-of-stream state then distinguishes a truncated stream from a
complete stream followed by trailing or concatenated data.

## When to Use

Use this recipe at a boundary that accepts one already materialized zlib stream
and must reject partial, concatenated, or unexpectedly expanding input. Both
limits should come from a local policy rather than from fields inside the
untrusted payload.

Use an incremental state machine when compressed bytes arrive as a stream, and
use the matching gzip, raw DEFLATE, or archive API when that is the declared
format. Authenticate data separately when a sender must be trusted; zlib's
checksum detects accidental corruption but is not a message authenticator.

## Implementation

```python
import zlib
from enum import StrEnum


_MAX_INPUT_LIMIT = 16 * 1024 * 1024
_MAX_OUTPUT_LIMIT = 64 * 1024 * 1024


class ZlibProblem(StrEnum):
    INPUT_LIMIT_EXCEEDED = "input_limit_exceeded"
    OUTPUT_LIMIT_EXCEEDED = "output_limit_exceeded"
    TRUNCATED_STREAM = "truncated_stream"
    TRAILING_OR_CONCATENATED_DATA = "trailing_or_concatenated_data"
    CORRUPT_STREAM = "corrupt_stream"


class ZlibStreamError(ValueError):
    def __init__(self, problem: ZlibProblem) -> None:
        self.problem = problem
        super().__init__(problem.value)


def _positive_limit(value: object, *, name: str, maximum: int) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not 1 <= value <= maximum:
        raise ValueError(f"{name} is outside the supported range")
    return value


def decompress_exact_zlib_stream(
    data: bytes,
    *,
    input_limit: int,
    output_limit: int,
) -> bytes:
    if type(data) is not bytes:
        raise TypeError("data must be exact bytes")
    checked_input_limit = _positive_limit(
        input_limit,
        name="input_limit",
        maximum=_MAX_INPUT_LIMIT,
    )
    checked_output_limit = _positive_limit(
        output_limit,
        name="output_limit",
        maximum=_MAX_OUTPUT_LIMIT,
    )
    if len(data) > checked_input_limit:
        raise ZlibStreamError(ZlibProblem.INPUT_LIMIT_EXCEEDED)

    decompressor = zlib.decompressobj()
    try:
        result = decompressor.decompress(data, checked_output_limit + 1)
    except zlib.error:
        raise ZlibStreamError(ZlibProblem.CORRUPT_STREAM) from None

    if len(result) > checked_output_limit or decompressor.unconsumed_tail:
        raise ZlibStreamError(ZlibProblem.OUTPUT_LIMIT_EXCEEDED)
    if not decompressor.eof:
        raise ZlibStreamError(ZlibProblem.TRUNCATED_STREAM)
    if decompressor.unused_data:
        raise ZlibStreamError(ZlibProblem.TRAILING_OR_CONCATENATED_DATA)
    return result
```

## Example

```python
payload = b"alpha\nbeta\n"
encoded = zlib.compress(payload)
decoded = decompress_exact_zlib_stream(
    encoded,
    input_limit=1_024,
    output_limit=len(payload),
)

corrupted = bytearray(encoded)
corrupted[-1] ^= 1


def observed_problem(
    value: bytes,
    *,
    input_limit: int = 1_024,
    output_limit: int = 1_024,
) -> ZlibProblem:
    try:
        decompress_exact_zlib_stream(
            value,
            input_limit=input_limit,
            output_limit=output_limit,
        )
    except ZlibStreamError as error:
        return error.problem
    raise AssertionError("the test value was unexpectedly accepted")


problems = (
    observed_problem(encoded, input_limit=len(encoded) - 1),
    observed_problem(zlib.compress(b"x" * 33), output_limit=32),
    observed_problem(encoded[:-1]),
    observed_problem(encoded + b"trailing"),
    observed_problem(encoded + zlib.compress(b"second")),
    observed_problem(bytes(corrupted)),
)

assert (decoded, problems) == (
    payload,
    (
        ZlibProblem.INPUT_LIMIT_EXCEEDED,
        ZlibProblem.OUTPUT_LIMIT_EXCEEDED,
        ZlibProblem.TRUNCATED_STREAM,
        ZlibProblem.TRAILING_OR_CONCATENATED_DATA,
        ZlibProblem.TRAILING_OR_CONCATENATED_DATA,
        ZlibProblem.CORRUPT_STREAM,
    ),
)
```

## Trade-offs and Limitations

The function materializes the complete compressed input, but its accepted size
is bounded before decompression. The returned allocation is limited to one byte
beyond `output_limit`; zlib still owns bounded internal state and consumes CPU
according to the admitted compressed and decompressed work. This is not a
general defense against every resource-exhaustion strategy.

Error precedence follows the first decisive policy violation. Input that
exceeds its compressed limit is not parsed. A stream that reaches the output
guard is rejected without inspecting later bytes, so later truncation or
corruption is intentionally not classified. `CORRUPT_STREAM` means only that
zlib reported a format or checksum error during admitted work. The recipe
accepts the zlib wrapper only; it does not accept raw DEFLATE or gzip, recover
partial output, authenticate content, or interpret the decompressed bytes.

## Related Snippets

<!-- catalog:related:start -->
- [Verify a Bounded Byte Stream Before Returning Its Payload](verify-a-bounded-byte-stream-before-returning-its-payload.md)
- [Build a Deterministic Size-Capped USTAR Archive from Bytes](../configuration-serialization/build-a-deterministic-size-capped-ustar-archive-from-bytes.md)
- [Release a Pooled Response Connection Only After Clean EOF](../networking-protocols/release-a-pooled-response-connection-only-after-clean-eof.md)
<!-- catalog:related:end -->
