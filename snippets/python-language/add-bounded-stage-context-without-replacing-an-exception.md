---
title: "Add Bounded Stage Context Without Replacing an Exception"
snippet_type: idiom
use_cases:
  - observability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - keep-exception-handlers-narrow-with-try-else.md
  - ../observability-operations/capture-a-bounded-pickle-friendly-exception-report.md
---

# Add Bounded Stage Context Without Replacing an Exception

## Idea and Problem

Add one bounded diagnostic note at a synchronous stage boundary, then re-raise the same exception without replacing its type or traceback chain.

Wrapping every failure in a new exception can obscure error handling that
depends on the original type. A context manager can instead use
`BaseException.add_note()` and a bare `raise`. A reserved note prefix makes
stage notes countable and deduplicated, while a small cap prevents nested
boundaries from growing diagnostics without limit.

## When to Use

Use this idiom around a small synchronous operation when the operation's stage
is useful diagnostic context and callers must still receive the original
exception. Stage labels must be stable, non-sensitive identifiers rather than
request data, paths, user input, or credentials. The context manager catches
only ordinary `Exception` instances; control-flow exceptions derived directly
from `BaseException` pass through without annotation.

Use exception translation when the public API intentionally changes the error
contract. Use a structured report when diagnostics must cross a process or
serialization boundary. Logging, retries, async cancellation, and pipeline
execution are separate responsibilities.

## Implementation

```python
import re
from collections.abc import Iterator
from contextlib import contextmanager


_MAX_STAGE_NOTES = 8
_STAGE_NOTE_PREFIX = "stage-context: "
_STAGE_NAME = re.compile(r"[a-z][a-z0-9_.-]{0,63}", re.ASCII)
_MISSING_NOTES = object()


def _validate_stage(value: object) -> str:
    if type(value) is not str:
        raise TypeError("stage must be an exact string")
    if _STAGE_NAME.fullmatch(value) is None:
        raise ValueError("stage must be a conservative ASCII identifier")
    return value


def _try_add_stage_note(error: Exception, note: str) -> None:
    try:
        try:
            notes = object.__getattribute__(error, "__notes__")
        except AttributeError:
            notes = _MISSING_NOTES

        if notes is _MISSING_NOTES:
            stage_notes: tuple[str, ...] = ()
        elif type(notes) is list:
            stage_notes = tuple(
                existing
                for existing in notes
                if type(existing) is str
                and existing.startswith(_STAGE_NOTE_PREFIX)
            )
        else:
            return

        if note in stage_notes or len(stage_notes) >= _MAX_STAGE_NOTES:
            return
        BaseException.add_note(error, note)
    except Exception:
        return


@contextmanager
def bounded_stage_context(stage: str) -> Iterator[None]:
    checked_stage = _validate_stage(stage)
    try:
        yield
    except Exception as error:
        _try_add_stage_note(
            error,
            f"{_STAGE_NOTE_PREFIX}{checked_stage}",
        )
        raise
```

## Example

```python
class DecodeFailure(RuntimeError):
    pass


raised_errors: list[DecodeFailure] = []


def decode_count(text: str) -> int:
    try:
        return int(text)
    except ValueError as cause:
        failure = DecodeFailure("count was rejected")
        raised_errors.append(failure)
        raise failure from cause


try:
    with bounded_stage_context("decode"):
        with bounded_stage_context("decode"):
            decode_count("many")
except DecodeFailure as caught:
    observed_error = caught
    traceback_names = []
    traceback = caught.__traceback__
    while traceback is not None:
        traceback_names.append(traceback.tb_frame.f_code.co_name)
        traceback = traceback.tb_next
else:
    raise AssertionError("expected DecodeFailure")


class StopNow(BaseException):
    pass


signal = StopNow()
try:
    with bounded_stage_context("decode"):
        raise signal
except StopNow as propagated_signal:
    observed_signal = propagated_signal
else:
    raise AssertionError("expected StopNow")


assert (
    observed_error is raised_errors[0],
    type(observed_error) is DecodeFailure,
    type(observed_error.__cause__) is ValueError,
    observed_error.__context__ is observed_error.__cause__,
    observed_error.__notes__ == ["stage-context: decode"],
    "decode_count" in traceback_names,
    observed_signal is signal,
    not hasattr(signal, "__notes__"),
) == (True, True, True, True, True, True, True, True)
```

## Trade-offs and Limitations

The exception object is preserved, but it is intentionally mutated by adding a
note. The helper adds at most eight notes with its reserved prefix and skips an
exact duplicate; unrelated notes added elsewhere are not bounded. If an
exception has a malformed note container or note attachment raises an ordinary
exception, annotation is skipped so the active failure is not masked.

Stage notes may appear in rendered tracebacks and logs, so even a syntactically
valid label must be non-sensitive. A bare `raise` preserves the existing type,
identity, cause, context, and traceback chain. This idiom does not capture
locals, serialize errors, log anything, wrap exceptions, or handle asynchronous
context managers.

## Related Snippets

<!-- catalog:related:start -->
- [Keep Exception Handlers Narrow with try/else](keep-exception-handlers-narrow-with-try-else.md)
- [Capture a Bounded Pickle-Friendly Exception Report](../observability-operations/capture-a-bounded-pickle-friendly-exception-report.md)
<!-- catalog:related:end -->
