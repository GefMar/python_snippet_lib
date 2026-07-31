---
title: "Decompress One Size-Capped Single Zstandard Frame"
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
  - decompress-exactly-one-zlib-stream-under-input-and-output-limits.md
  - verify-a-bounded-byte-stream-before-returning-its-payload.md
  - ../configuration-serialization/create-reproducible-gzip-bytes-with-an-explicit-zero-modification-time.md
---

# Decompress One Size-Capped Single Zstandard Frame

## Idea and Problem

Decompress one complete ordinary Zstandard frame only after bounding its encoded input, declared output, dictionary use, and decoder window.

The ordinary four-byte frame magic excludes skippable frames before structural
inspection. `get_frame_size()` must identify the whole input as exactly one
frame, while `get_frame_info()` must expose a known decompressed size within the
caller's limit and no recorded dictionary ID.

An incremental `ZstdDecompressor` receives at most the remaining allowance up
to `output_limit + 1`. Its terminal state must confirm end of frame, no need for
more input, no unused bytes, and an output length equal to the declared size.

## When to Use

Use this recipe at an in-memory boundary whose protocol admits one ordinary
Zstandard frame with a recorded content size. Choose the input, output, and
window-log limits from local policy before examining untrusted data. A smaller
`window_log_max` rejects frames whose decoding window exceeds that policy even
when their declared output would fit.

Use a separately designed incremental input boundary when compressed bytes
arrive from a stream. If a protocol legitimately uses skippable frames,
multiple frames, unknown content sizes, or dictionaries, model those features
explicitly instead of weakening this single-frame profile.

## Implementation

```python
from compression import zstd
from enum import StrEnum

_FRAME_MAGIC = b"\x28\xb5\x2f\xfd"
_INPUT_CHUNK_BYTES = 64 * 1024
_MAX_INPUT_LIMIT = 16 * 1024 * 1024
_MAX_OUTPUT_LIMIT = 64 * 1024 * 1024
_MAX_WINDOW_LOG = 26


class ZstandardFrameProblem(StrEnum):
    INPUT_LIMIT_EXCEEDED = "input_limit_exceeded"
    NOT_AN_ORDINARY_FRAME = "not_an_ordinary_frame"
    INVALID_FRAME = "invalid_frame"
    TRAILING_OR_CONCATENATED_DATA = "trailing_or_concatenated_data"
    UNKNOWN_DECOMPRESSED_SIZE = "unknown_decompressed_size"
    OUTPUT_LIMIT_EXCEEDED = "output_limit_exceeded"
    RECORDED_DICTIONARY_ID = "recorded_dictionary_id"
    DECOMPRESSION_FAILED = "decompression_failed"
    INCOMPLETE_FRAME = "incomplete_frame"
    INVALID_DECOMPRESSOR_STATE = "invalid_decompressor_state"
    DECLARED_SIZE_MISMATCH = "declared_size_mismatch"


class ZstandardFrameError(ValueError):
    def __init__(self, problem: ZstandardFrameProblem) -> None:
        self.problem = problem
        super().__init__(problem.value)


def _exact_limit(
    value: object,
    *,
    name: str,
    minimum: int,
    maximum: int,
) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not minimum <= value <= maximum:
        raise ValueError(f"{name} is outside the supported range")
    return value


def decompress_one_zstandard_frame(
    data: bytes,
    *,
    input_limit: int,
    output_limit: int,
    window_log_max: int,
) -> bytes:
    if type(data) is not bytes:
        raise TypeError("data must be exact bytes")

    checked_input_limit = _exact_limit(
        input_limit,
        name="input_limit",
        minimum=1,
        maximum=_MAX_INPUT_LIMIT,
    )
    checked_output_limit = _exact_limit(
        output_limit,
        name="output_limit",
        minimum=0,
        maximum=_MAX_OUTPUT_LIMIT,
    )
    api_window_min, api_window_max = zstd.DecompressionParameter.window_log_max.bounds()
    checked_window_log_max = _exact_limit(
        window_log_max,
        name="window_log_max",
        minimum=api_window_min,
        maximum=min(api_window_max, _MAX_WINDOW_LOG),
    )

    if len(data) > checked_input_limit:
        raise ZstandardFrameError(ZstandardFrameProblem.INPUT_LIMIT_EXCEEDED)
    if not data.startswith(_FRAME_MAGIC):
        raise ZstandardFrameError(ZstandardFrameProblem.NOT_AN_ORDINARY_FRAME)

    try:
        frame_size = zstd.get_frame_size(data)
    except zstd.ZstdError:
        raise ZstandardFrameError(ZstandardFrameProblem.INVALID_FRAME) from None
    if frame_size != len(data):
        raise ZstandardFrameError(
            ZstandardFrameProblem.TRAILING_OR_CONCATENATED_DATA,
        )

    try:
        frame_info = zstd.get_frame_info(data)
    except zstd.ZstdError:
        raise ZstandardFrameError(ZstandardFrameProblem.INVALID_FRAME) from None
    declared_size = frame_info.decompressed_size
    if declared_size is None:
        raise ZstandardFrameError(
            ZstandardFrameProblem.UNKNOWN_DECOMPRESSED_SIZE,
        )
    if declared_size > checked_output_limit:
        raise ZstandardFrameError(ZstandardFrameProblem.OUTPUT_LIMIT_EXCEEDED)
    if frame_info.dictionary_id != 0:
        raise ZstandardFrameError(
            ZstandardFrameProblem.RECORDED_DICTIONARY_ID,
        )

    try:
        decompressor = zstd.ZstdDecompressor(
            options={
                zstd.DecompressionParameter.window_log_max: checked_window_log_max,
            },
        )
    except zstd.ZstdError:
        raise ZstandardFrameError(
            ZstandardFrameProblem.DECOMPRESSION_FAILED,
        ) from None

    output_parts: list[bytes] = []
    produced = 0
    input_offset = 0
    output_allowance = checked_output_limit + 1

    while not decompressor.eof and produced < output_allowance:
        if decompressor.needs_input:
            if input_offset == len(data):
                break
            input_chunk = data[input_offset : input_offset + _INPUT_CHUNK_BYTES]
            input_offset += len(input_chunk)
        else:
            input_chunk = b""

        try:
            output_chunk = decompressor.decompress(
                input_chunk,
                output_allowance - produced,
            )
        except zstd.ZstdError:
            raise ZstandardFrameError(
                ZstandardFrameProblem.DECOMPRESSION_FAILED,
            ) from None
        output_parts.append(output_chunk)
        produced += len(output_chunk)

        if (
            not output_chunk
            and not input_chunk
            and not decompressor.eof
            and not decompressor.needs_input
        ):
            raise ZstandardFrameError(
                ZstandardFrameProblem.INVALID_DECOMPRESSOR_STATE,
            )

    if produced > checked_output_limit:
        raise ZstandardFrameError(ZstandardFrameProblem.OUTPUT_LIMIT_EXCEEDED)
    if not decompressor.eof:
        if not decompressor.needs_input:
            raise ZstandardFrameError(
                ZstandardFrameProblem.OUTPUT_LIMIT_EXCEEDED,
            )
        raise ZstandardFrameError(ZstandardFrameProblem.INCOMPLETE_FRAME)
    if decompressor.needs_input:
        raise ZstandardFrameError(
            ZstandardFrameProblem.INVALID_DECOMPRESSOR_STATE,
        )
    if input_offset != len(data) or decompressor.unused_data:
        raise ZstandardFrameError(
            ZstandardFrameProblem.TRAILING_OR_CONCATENATED_DATA,
        )

    result = b"".join(output_parts)
    if len(result) != declared_size:
        raise ZstandardFrameError(
            ZstandardFrameProblem.DECLARED_SIZE_MISMATCH,
        )
    return result


```

## Example

```python
frame = bytes.fromhex("28b52ffd240b590000616c7068610a626574610a746c5207")
decoded = decompress_one_zstandard_frame(
    frame,
    input_limit=1_024,
    output_limit=11,
    window_log_max=20,
)


def observed_problem(
    value: bytes,
    *,
    input_limit: int = 1_024,
    output_limit: int = 64,
) -> ZstandardFrameProblem:
    try:
        decompress_one_zstandard_frame(
            value,
            input_limit=input_limit,
            output_limit=output_limit,
            window_log_max=20,
        )
    except ZstandardFrameError as error:
        return error.problem
    raise AssertionError("the test frame was unexpectedly accepted")


unknown_size_frame = bytes.fromhex("28b52ffd000045000010787801003f012c")
skippable_frame = bytes.fromhex("502a4d1800000000")
corrupted_frame = frame[:-1] + bytes((frame[-1] ^ 1,))
problems = (
    observed_problem(frame, input_limit=len(frame) - 1),
    observed_problem(frame, output_limit=10),
    observed_problem(frame[:-1]),
    observed_problem(frame + b"trailing"),
    observed_problem(frame + frame),
    observed_problem(skippable_frame),
    observed_problem(unknown_size_frame),
    observed_problem(corrupted_frame),
)

assert (decoded, problems) == (
    b"alpha\nbeta\n",
    (
        ZstandardFrameProblem.INPUT_LIMIT_EXCEEDED,
        ZstandardFrameProblem.OUTPUT_LIMIT_EXCEEDED,
        ZstandardFrameProblem.INVALID_FRAME,
        ZstandardFrameProblem.TRAILING_OR_CONCATENATED_DATA,
        ZstandardFrameProblem.TRAILING_OR_CONCATENATED_DATA,
        ZstandardFrameProblem.NOT_AN_ORDINARY_FRAME,
        ZstandardFrameProblem.UNKNOWN_DECOMPRESSED_SIZE,
        ZstandardFrameProblem.DECOMPRESSION_FAILED,
    ),
)
```

## Trade-offs and Limitations

The complete compressed input is materialized and bounded before parsing.
Frame-size inspection scans bounded input, and decompression can produce at
most `output_limit + 1` bytes before rejection. The result fragments and final
joined `bytes` value can temporarily require roughly twice the admitted output
space. Input size, output size, and the decoder window are bounded, but CPU
work is still determined by the admitted frame.

The profile rejects skippable frames, multiple frames, trailing bytes, unknown
content sizes, and every nonzero recorded dictionary ID. Dictionary ID zero
means only that no ID was recorded; a frame might still require unspecified
dictionary content, but this function supplies no dictionary and rejects the
resulting decoder failure. A window-policy rejection, malformed compressed
content, a checksum failure, and an unrecorded missing dictionary all map to
`DECOMPRESSION_FAILED` because the decoder does not expose a stable typed
distinction between them.

Zstandard frames do not have to carry a checksum, so successful decompression
does not prove authenticity or detect every accidental modification. Verify a
trusted digest or authenticator separately when content identity matters. This
recipe does not accept a stream source, compress data, train dictionaries,
recover partial output, or interpret the returned bytes.

## Related Snippets

<!-- catalog:related:start -->
- [Decompress Exactly One Zlib Stream Under Input and Output Limits](decompress-exactly-one-zlib-stream-under-input-and-output-limits.md)
- [Verify a Bounded Byte Stream Before Returning Its Payload](verify-a-bounded-byte-stream-before-returning-its-payload.md)
- [Create Reproducible gzip Bytes with an Explicit Zero Modification Time](../configuration-serialization/create-reproducible-gzip-bytes-with-an-explicit-zero-modification-time.md)
<!-- catalog:related:end -->
