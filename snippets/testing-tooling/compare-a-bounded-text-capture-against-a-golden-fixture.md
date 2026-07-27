---
title: "Compare a Bounded Text Capture Against a Golden Fixture"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - extract-bounded-native-test-failure-highlights.md
  - ../configuration-serialization/render-a-stable-unified-diff-for-nested-json-values.md
  - ../concurrency-lifecycle/stream-bounded-stdout-and-stderr-lines-from-a-posix-process.md
---

# Compare a Bounded Text Capture Against a Golden Fixture

## Idea and Problem

Compare an already captured exit status and text streams with an exact golden record while keeping mismatch diagnostics deterministic and bounded.

Separating capture from comparison makes the oracle safe to exercise without a
real command. The comparison accepts only canonical UTF-8 text with LF newlines,
checks status, stdout, and stderr in a fixed order, and returns immutable
mismatch values instead of updating a fixture or raising an opaque assertion.

## When to Use

Use this technique when a small CLI, compiler, formatter, or native adapter has
a stable human-readable output contract whose complete shape matters. Capture
the process separately with explicit time, byte, environment, and working
directory limits, then review and store the small expected record as text.

Prefer focused assertions for critical semantics and for values that change
frequently. Normalize timestamps, paths, ordering, or platform details before
this boundary only when that normalization is part of the tested contract. Do
not add a fixture-update mode to the comparison used in ordinary test runs.

## Implementation

```python
import difflib
from dataclasses import dataclass


_MAX_STREAM_BYTES = 64 * 1024
_MAX_STREAM_LINES = 2_000
_MAX_DIFF_LINES = 100
_MAX_DIFF_CHARACTERS = 8_192


@dataclass(frozen=True, slots=True)
class TextCapture:
    exit_code: int
    stdout: str
    stderr: str


@dataclass(frozen=True, slots=True)
class CaptureMismatch:
    field: str
    detail: str


@dataclass(frozen=True, slots=True)
class GoldenComparison:
    matches: bool
    mismatches: tuple[CaptureMismatch, ...]


def _validated_exit_code(value: object) -> int:
    if isinstance(value, bool) or not isinstance(value, int):
        raise TypeError("exit_code must be an integer")
    if not -255 <= value <= 255:
        raise ValueError("exit_code is outside the supported range")
    return value


def _validated_text(value: object, *, name: str) -> str:
    if not isinstance(value, str):
        raise TypeError(f"{name} must be text")
    if "\r" in value:
        raise ValueError(f"{name} must use LF newlines")
    if len(value.splitlines()) > _MAX_STREAM_LINES:
        raise ValueError(f"{name} has too many lines")
    if len(value.encode("utf-8")) > _MAX_STREAM_BYTES:
        raise ValueError(f"{name} exceeds the supported byte size")
    if any(character not in "\n\t" and not character.isprintable() for character in value):
        raise ValueError(f"{name} contains unsupported control characters")
    return value


def _validate_limit(value: object, *, name: str, upper: int) -> int:
    if isinstance(value, bool) or not isinstance(value, int):
        raise TypeError(f"{name} must be an integer")
    if not 1 <= value <= upper:
        raise ValueError(f"{name} is outside the supported range")
    return value


def _bounded_unified_diff(
    expected: str,
    actual: str,
    *,
    field: str,
    maximum_lines: int,
    maximum_characters: int,
) -> str:
    def diff_lines(text: str) -> list[str]:
        lines = text.splitlines(keepends=True)
        if text and not text.endswith("\n"):
            lines[-1] += "\n"
            lines.append("\\ No newline at end of capture\n")
        return lines

    diff = difflib.unified_diff(
        diff_lines(expected),
        diff_lines(actual),
        fromfile=f"expected/{field}",
        tofile=f"actual/{field}",
        n=2,
        lineterm="\n",
    )
    selected: list[str] = []
    selected_characters = 0
    truncated = False
    for line in diff:
        if (
            len(selected) >= maximum_lines
            or selected_characters + len(line) > maximum_characters
        ):
            truncated = True
            break
        selected.append(line)
        selected_characters += len(line)

    if truncated:
        marker = "... [diff truncated]\n"
        while selected and (
            len(selected) >= maximum_lines
            or selected_characters + len(marker) > maximum_characters
        ):
            selected_characters -= len(selected.pop())
        remaining_characters = maximum_characters - selected_characters
        selected.append(marker[:remaining_characters])
    return "".join(selected)


def compare_with_golden(
    actual: TextCapture,
    expected: TextCapture,
    *,
    maximum_diff_lines: int = 40,
    maximum_diff_characters: int = 4_096,
) -> GoldenComparison:
    if not isinstance(actual, TextCapture) or not isinstance(expected, TextCapture):
        raise TypeError("actual and expected must be TextCapture values")
    maximum_diff_lines = _validate_limit(
        maximum_diff_lines,
        name="maximum_diff_lines",
        upper=_MAX_DIFF_LINES,
    )
    maximum_diff_characters = _validate_limit(
        maximum_diff_characters,
        name="maximum_diff_characters",
        upper=_MAX_DIFF_CHARACTERS,
    )

    actual_code = _validated_exit_code(actual.exit_code)
    expected_code = _validated_exit_code(expected.exit_code)
    actual_stdout = _validated_text(actual.stdout, name="actual stdout")
    expected_stdout = _validated_text(expected.stdout, name="expected stdout")
    actual_stderr = _validated_text(actual.stderr, name="actual stderr")
    expected_stderr = _validated_text(expected.stderr, name="expected stderr")

    mismatches: list[CaptureMismatch] = []
    if actual_code != expected_code:
        mismatches.append(
            CaptureMismatch(
                "exit_code",
                f"expected {expected_code}, got {actual_code}",
            )
        )
    for field, expected_text, actual_text in (
        ("stdout", expected_stdout, actual_stdout),
        ("stderr", expected_stderr, actual_stderr),
    ):
        if actual_text != expected_text:
            mismatches.append(
                CaptureMismatch(
                    field,
                    _bounded_unified_diff(
                        expected_text,
                        actual_text,
                        field=field,
                        maximum_lines=maximum_diff_lines,
                        maximum_characters=maximum_diff_characters,
                    ),
                )
            )

    result = tuple(mismatches)
    return GoldenComparison(matches=not result, mismatches=result)
```

## Example

```python
golden = TextCapture(
    exit_code=0,
    stdout="items: 2\nstatus: ready\n",
    stderr="",
)
same = compare_with_golden(golden, golden)
changed = compare_with_golden(
    TextCapture(2, "items: 3\nstatus: ready\n", "invalid input\n"),
    golden,
)
missing_final_newline = compare_with_golden(
    TextCapture(0, "items: 3", ""),
    TextCapture(0, "items: 2", ""),
)

assert (
    same.matches,
    changed.matches,
    tuple(mismatch.field for mismatch in changed.mismatches),
    "-items: 2" in changed.mismatches[1].detail,
    "No newline at end of capture" in missing_final_newline.mismatches[0].detail,
) == (True, False, ("exit_code", "stdout", "stderr"), True, True)
```

## Trade-offs and Limitations

Golden comparisons catch broad formatting regressions but often explain less
than focused behavioral assertions. A large or frequently changing fixture is
expensive to review and encourages blind updates. Keep captures small, review
fixture changes as code changes, and add direct tests for important branches.

This helper neither launches nor isolates a process. The capture layer still
owns timeouts, output caps, environment minimization, cleanup, and decoding.
The returned diff contains bounded portions of both streams and therefore is
not automatically safe for secrets; redact sensitive values before comparison.
Exact LF text comparison also makes platform-specific newline conversion an
explicit responsibility of the capture contract.

## Related Snippets

<!-- catalog:related:start -->
- [Extract Bounded Native-Test Failure Highlights](extract-bounded-native-test-failure-highlights.md)
- [Render a Stable Unified Diff for Nested JSON Values](../configuration-serialization/render-a-stable-unified-diff-for-nested-json-values.md)
- [Stream Bounded stdout and stderr Lines from a POSIX Process](../concurrency-lifecycle/stream-bounded-stdout-and-stderr-lines-from-a-posix-process.md)
<!-- catalog:related:end -->
