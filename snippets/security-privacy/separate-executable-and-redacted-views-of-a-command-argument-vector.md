---
title: "Separate Executable and Redacted Views of a Command Argument Vector"
snippet_type: pattern
use_cases:
  - observability
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - redact-a-printable-ascii-secret-with-a-bounded-visible-tail.md
  - ../observability-operations/format-log-records-as-json-with-explicit-extra-fields.md
  - ../concurrency-lifecycle/stream-bounded-stdout-and-stderr-lines-from-a-posix-process.md
---

# Separate Executable and Redacted Views of a Command Argument Vector

## Idea and Problem

Build one immutable command plan that keeps the shell-free execution vector hidden from representation while exposing a separately rendered view with fixed secret placeholders.

Structural secret markers avoid trying to rediscover sensitive substrings after
the command is assembled. Public arguments must be printable and safe for one
log line; secret values may contain arbitrary text except NUL but are never
passed to the display renderer.

## When to Use

Use this pattern when a subprocess API unavoidably requires a credential in one
argument and application logs still need a useful command shape. Construct the
plan immediately before a trusted runner receives its immutable tuple, and log
only the `display` field as a structured parameter.

Prefer standard input, a protected file descriptor, an inherited credential
store, or the program's documented secret channel. Keep all ordinary arguments
non-sensitive: marking one value secret does not sanitize neighboring values,
the child environment, output, exceptions, or operating-system process views.

## Implementation

```python
import shlex
from collections.abc import Iterable
from dataclasses import dataclass, field
from itertools import islice


_MAX_ARGUMENTS = 128
_MAX_ARGUMENT_BYTES = 4_096
_MAX_TOTAL_ARGUMENT_BYTES = 32 * 1024
_MAX_DISPLAY_CHARACTERS = 64 * 1024
_SECRET_PLACEHOLDER = "<redacted>"


@dataclass(frozen=True, slots=True)
class SecretArgument:
    value: str = field(repr=False)


@dataclass(frozen=True, slots=True)
class CommandPlan:
    _execution_argv: tuple[str, ...] = field(repr=False)
    display: str

    def execution_argv(self) -> tuple[str, ...]:
        return self._execution_argv


def _validated_argument(value: object, *, secret: bool) -> str:
    if not isinstance(value, str):
        raise TypeError("command arguments must be text")
    if "\x00" in value:
        raise ValueError("command arguments must not contain NUL")
    if secret and not value:
        raise ValueError("secret arguments must not be empty")
    if not secret and any(not character.isprintable() for character in value):
        raise ValueError("public arguments must contain printable characters only")
    if len(value.encode("utf-8")) > _MAX_ARGUMENT_BYTES:
        raise ValueError("a command argument exceeds the supported byte size")
    return value


def build_command_plan(
    parts: Iterable[str | SecretArgument],
) -> CommandPlan:
    if isinstance(parts, (str, bytes)):
        raise TypeError("parts must be a non-text iterable")
    snapshot = tuple(islice(parts, _MAX_ARGUMENTS + 1))
    if not 1 <= len(snapshot) <= _MAX_ARGUMENTS:
        raise ValueError("argument count is outside the supported range")
    if isinstance(snapshot[0], SecretArgument):
        raise ValueError("the executable name must be a public argument")

    execution: list[str] = []
    display_parts: list[str] = []
    total_argument_bytes = 0
    for part in snapshot:
        if isinstance(part, SecretArgument):
            value = _validated_argument(part.value, secret=True)
            display_value = _SECRET_PLACEHOLDER
        elif isinstance(part, str):
            value = _validated_argument(part, secret=False)
            display_value = value
        else:
            raise TypeError("parts must contain text or SecretArgument values")
        total_argument_bytes += len(value.encode("utf-8"))
        if total_argument_bytes > _MAX_TOTAL_ARGUMENT_BYTES:
            raise ValueError("command arguments exceed the aggregate byte limit")
        execution.append(value)
        display_parts.append(display_value)

    if not execution[0]:
        raise ValueError("the executable name must not be empty")
    display = shlex.join(display_parts)
    if len(display) > _MAX_DISPLAY_CHARACTERS:
        raise ValueError("redacted display exceeds the supported size")
    return CommandPlan(tuple(execution), display)
```

## Example

```python
secret = SecretArgument('line-one\n"token-value"')
plan = build_command_plan(
    (
        "uploader",
        "--credential",
        secret,
        "--label",
        "sample value",
    )
)

visible = plan.display + repr(plan) + repr(secret)
assert (
    plan.execution_argv(),
    plan.display,
    "token-value" in visible,
) == (
    ("uploader", "--credential", 'line-one\n"token-value"', "--label", "sample value"),
    "uploader --credential '<redacted>' --label 'sample value'",
    False,
)
```

## Trade-offs and Limitations

The hidden tuple still contains the real value and can be retrieved by trusted
code. Passing it to a child may expose it through process inspection, audit
events, crash reports, or child output. Immutable Python strings also cannot be
reliably erased from memory. This is a log-shaping boundary, not a secret
transport or operating-system isolation mechanism.

The helper does not execute a process, choose an environment, capture output,
set a timeout, or clean up children. It also cannot detect a secret accidentally
supplied as an ordinary string. Public arguments are intentionally restricted
to printable single-line values so the rendered display cannot inject terminal
controls or extra log lines. `shlex.join` produces a POSIX-style diagnostic
view only; never pass the rendered string to a shell or treat it as portable
process-launch syntax.

## Related Snippets

<!-- catalog:related:start -->
- [Redact a Printable ASCII Secret with a Bounded Visible Tail](redact-a-printable-ascii-secret-with-a-bounded-visible-tail.md)
- [Format Log Records as JSON with Explicit Extra Fields](../observability-operations/format-log-records-as-json-with-explicit-extra-fields.md)
- [Stream Bounded stdout and stderr Lines from a POSIX Process](../concurrency-lifecycle/stream-bounded-stdout-and-stderr-lines-from-a-posix-process.md)
<!-- catalog:related:end -->
