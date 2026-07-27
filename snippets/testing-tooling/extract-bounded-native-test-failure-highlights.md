---
title: "Extract Bounded Native-Test Failure Highlights"
snippet_type: recipe
use_cases:
  - observability
  - parsing
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - ../concurrency-lifecycle/stream-bounded-stdout-and-stderr-lines-from-a-posix-process.md
  - ../data-processing/limit-text-lines-across-arbitrary-chunks.md
  - ../observability-operations/capture-a-bounded-pickle-friendly-exception-report.md
---

# Extract Bounded Native-Test Failure Highlights

## Idea and Problem

Extract a small, attributed set of diagnostic lines from already bounded native-test stdout and stderr without treating the result as a complete report.

Native test runners and sanitizers often surround a useful failure line with a
large amount of unrelated output. Explicit byte, line, line-length, match, and
report limits keep extraction predictable. Stream names and original line
numbers preserve enough context to locate the selected diagnostics.

## When to Use

Use this recipe after a subprocess runner has already captured a bounded amount
of stdout and stderr. Supply a short set of stable, case-insensitive markers
for the formats you operate and a redaction callback appropriate for the data.
It is useful for concise CI summaries or structured failure records where the
complete bounded logs remain available separately.

Do not use marker matching to decide whether a test passed, to parse an
unbounded stream, or to claim that every failure was found. Exit status and the
test protocol remain authoritative. Redaction is caller-owned because generic
code cannot know which paths, identifiers, or values are sensitive.

## Implementation

```python
import io
from collections.abc import Callable, Sequence
from dataclasses import dataclass


_MAX_STREAM_BYTES = 256 * 1024
_MAX_LINES_PER_STREAM = 4_096
_MAX_LINE_BYTES = 4_096
_MAX_MARKERS = 16
_MAX_MARKER_CHARACTERS = 64
_MAX_HIGHLIGHTS = 64
_MAX_REPORT_CHARACTERS = 8_192


@dataclass(frozen=True, slots=True)
class FailureHighlight:
    stream: str
    line_number: int
    text: str
    clipped: bool


@dataclass(frozen=True, slots=True)
class FailureHighlights:
    highlights: tuple[FailureHighlight, ...]
    scanned_lines: int
    omitted_matches: int
    limited_streams: tuple[str, ...]


def _sanitize_controls(text: str) -> str:
    return "".join(
        " " if character == "\t" else character if character.isprintable() else "�"
        for character in text
    )


def extract_failure_highlights(
    stdout: bytes,
    stderr: bytes,
    *,
    markers: Sequence[str],
    redact: Callable[[str], str],
    max_lines_per_stream: int = 1_000,
    max_line_bytes: int = 1_024,
    max_highlights: int = 20,
    max_report_characters: int = 4_000,
) -> FailureHighlights:
    for name, value in (("stdout", stdout), ("stderr", stderr)):
        if not isinstance(value, bytes):
            raise TypeError(f"{name} must be immutable bytes")
        if len(value) > _MAX_STREAM_BYTES:
            raise ValueError(f"{name} exceeds the supported byte limit")
    if not isinstance(markers, Sequence) or isinstance(markers, str):
        raise TypeError("markers must be a non-text sequence")
    if not 1 <= len(markers) <= _MAX_MARKERS:
        raise ValueError("marker count is outside the supported range")
    if not callable(redact):
        raise TypeError("redact must be callable")

    for name, value, upper in (
        ("max_lines_per_stream", max_lines_per_stream, _MAX_LINES_PER_STREAM),
        ("max_line_bytes", max_line_bytes, _MAX_LINE_BYTES),
        ("max_highlights", max_highlights, _MAX_HIGHLIGHTS),
        (
            "max_report_characters",
            max_report_characters,
            _MAX_REPORT_CHARACTERS,
        ),
    ):
        if isinstance(value, bool) or not isinstance(value, int):
            raise TypeError(f"{name} must be an integer")
        if not 1 <= value <= upper:
            raise ValueError(f"{name} is outside the supported range")

    normalized_markers: list[str] = []
    for marker in markers:
        if not isinstance(marker, str):
            raise TypeError("markers must contain text")
        if (
            not 1 <= len(marker) <= _MAX_MARKER_CHARACTERS
            or not marker.isprintable()
        ):
            raise ValueError("a marker is outside the supported format")
        normalized_markers.append(marker.casefold())
    if len(set(normalized_markers)) != len(normalized_markers):
        raise ValueError("markers must be unique after case folding")

    highlights: list[FailureHighlight] = []
    scanned_lines = 0
    omitted_matches = 0
    report_characters = 0
    limited_streams: list[str] = []

    for stream_name, data in (("stdout", stdout), ("stderr", stderr)):
        buffer = io.BytesIO(data)
        line_number = 0
        while line_number < max_lines_per_stream:
            raw_line = buffer.readline()
            if not raw_line:
                break
            line_number += 1
            scanned_lines += 1

            content = raw_line.rstrip(b"\r\n")
            clipped = len(content) > max_line_bytes
            if clipped and stream_name not in limited_streams:
                limited_streams.append(stream_name)
            decoded = content[:max_line_bytes].decode("utf-8", errors="replace")
            sanitized = _sanitize_controls(decoded)
            folded = sanitized.casefold()
            if not any(marker in folded for marker in normalized_markers):
                continue

            remaining_characters = max_report_characters - report_characters
            if len(highlights) >= max_highlights or remaining_characters <= 0:
                omitted_matches += 1
                continue

            redacted = redact(sanitized)
            if not isinstance(redacted, str):
                raise TypeError("redact must return text")
            if len(redacted) > remaining_characters:
                redacted = redacted[:remaining_characters]
                clipped = True
            redacted = _sanitize_controls(redacted)
            highlights.append(
                FailureHighlight(
                    stream=stream_name,
                    line_number=line_number,
                    text=redacted,
                    clipped=clipped,
                )
            )
            report_characters += len(redacted)

        if buffer.read(1) and stream_name not in limited_streams:
            limited_streams.append(stream_name)

    return FailureHighlights(
        highlights=tuple(highlights),
        scanned_lines=scanned_lines,
        omitted_matches=omitted_matches,
        limited_streams=tuple(limited_streams),
    )
```

## Example

```python
stdout = b"running native checks\n2 checks completed\n"
stderr = (
    b"/build/worker/math_test.cpp:41: error: expected 8, received 5\n"
    b"test process stopped\n"
)

summary = extract_failure_highlights(
    stdout,
    stderr,
    markers=("error:", "assertion failed", "sanitizer"),
    redact=lambda line: line.replace("/build/worker/", "<build>/"),
)

assert summary == FailureHighlights(
    highlights=(
        FailureHighlight(
            stream="stderr",
            line_number=1,
            text="<build>/math_test.cpp:41: error: expected 8, received 5",
            clipped=False,
        ),
    ),
    scanned_lines=4,
    omitted_matches=0,
    limited_streams=(),
)
```

## Trade-offs and Limitations

Marker rules are deliberately simple and predictable, but they can miss
multiline diagnostics, localized output, or a format change, and they can match
ordinary text by accident. Scanning stops at the configured line limit, while
the report separately stops at its match or character limit. A clipped line or
listed limited stream tells the consumer to consult the retained bounded logs.

The redaction callback is not a security boundary: a missing rule can still
expose a secret, and a faulty callback can raise or expand text before it is
clipped. Run extraction only on data already suitable for the report's
audience. This helper does not launch processes, retain full output, parse
stack traces, or merge stdout and stderr by event time.

## Related Snippets

<!-- catalog:related:start -->
- [Stream Bounded stdout and stderr Lines from a POSIX Process](../concurrency-lifecycle/stream-bounded-stdout-and-stderr-lines-from-a-posix-process.md)
- [Limit Text Lines Across Arbitrary Chunks](../data-processing/limit-text-lines-across-arbitrary-chunks.md)
- [Capture a Bounded Pickle-Friendly Exception Report](../observability-operations/capture-a-bounded-pickle-friendly-exception-report.md)
<!-- catalog:related:end -->
