---
title: "Classify Bounded Interactive Python Source as Complete, Incomplete, or Invalid"
snippet_type: testing-technique
use_cases:
  - testing
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - audit-bounded-python-source-for-tab-width-independent-indentation.md
  - compare-bounded-python-expressions-structurally-without-execution.md
  - index-bounded-python-scope-bindings-with-symtable-without-execution.md
---

# Classify Bounded Interactive Python Source as Complete, Incomplete, or Invalid

## Idea and Problem

Classify bounded interactive Python source as complete, incomplete, or invalid with the single-input compiler grammar while discarding bytecode and never executing it.

This is different from parsing a complete module. An open suite, parenthesized
expression, or triple-quoted string is incomplete rather than invalid, and a
compound statement in `single` mode needs its terminating newline. Compilation
creates bytecode but does not run it.

## When to Use

Use this at a bounded interactive-input boundary for a console, notebook-like
cell editor, debugger prompt, or test of incremental source collection. It is
appropriate when matching the current CPython interactive grammar matters and
the caller needs a small status rather than a reusable code object.

Do not use it to authorize code, validate imports, inspect side effects, or
predict runtime success. Use `ast`, `symtable`, or a dedicated parser when the
caller needs structure. Execute accepted source only in a separately designed
and explicitly trusted environment.

## Implementation

```python
import warnings
from codeop import compile_command
from enum import StrEnum
from io import StringIO

_MAX_INTERACTIVE_CHARACTERS = 65_536
_MAX_INTERACTIVE_BYTES = 65_536
_MAX_INTERACTIVE_LINES = 4_096
_MAX_INTERACTIVE_LINE_CHARACTERS = 4_096


class InteractiveSourceLimitError(ValueError):
    """Raised when interactive source exceeds the bounded compiler profile."""


class InteractiveSourceStatus(StrEnum):
    COMPLETE = "complete"
    INCOMPLETE = "incomplete"
    INVALID = "invalid"


def _validate_interactive_source(source: str) -> None:
    if type(source) is not str:
        raise TypeError("source must be an exact str")
    if len(source) > _MAX_INTERACTIVE_CHARACTERS:
        raise InteractiveSourceLimitError("source exceeds the character limit")

    try:
        encoded = source.encode("utf-8")
    except UnicodeEncodeError as exc:
        raise InteractiveSourceLimitError("source must be valid UTF-8 text") from exc
    if len(encoded) > _MAX_INTERACTIVE_BYTES:
        raise InteractiveSourceLimitError("source exceeds the UTF-8 byte limit")

    physical_lines = tuple(StringIO(source))
    if len(physical_lines) > _MAX_INTERACTIVE_LINES:
        raise InteractiveSourceLimitError("source exceeds the physical-line limit")
    if any(
        len(line) > _MAX_INTERACTIVE_LINE_CHARACTERS for line in physical_lines
    ):
        raise InteractiveSourceLimitError(
            "a physical line exceeds the character limit"
        )


def classify_interactive_source(source: str) -> InteractiveSourceStatus:
    """Classify one bounded CPython `single` input without executing it."""
    _validate_interactive_source(source)

    try:
        with warnings.catch_warnings(record=True):
            warnings.simplefilter("always")
            compiled = compile_command(source, "<input>", "single")
    except (MemoryError, RecursionError) as exc:
        raise InteractiveSourceLimitError(
            "the compiler exceeded the supported resource profile"
        ) from exc
    except (SyntaxError, OverflowError, ValueError):
        return InteractiveSourceStatus.INVALID

    if compiled is None:
        return InteractiveSourceStatus.INCOMPLETE
    return InteractiveSourceStatus.COMPLETE
```

## Example

```python
complete = (
    classify_interactive_source("value = 1\n"),
    classify_interactive_source("1 + 2\n"),
    classify_interactive_source("if True:\n    value = 1\n"),
)
incomplete = (
    classify_interactive_source("if True:\n"),
    classify_interactive_source("if True:\n    value = 1"),
    classify_interactive_source("(1 +\n"),
    classify_interactive_source("'''open\n"),
)
invalid = classify_interactive_source("if True print('invalid')\n")
not_executed = classify_interactive_source("raise RuntimeError('must not run')\n")

filters_before = warnings.filters
showwarning_before = warnings.showwarning
warning_case = classify_interactive_source("1 is 1\n")
warning_state_restored = (
    warnings.filters is filters_before and warnings.showwarning is showwarning_before
)

line_limit_ok = classify_interactive_source("#" + "x" * 4_095)
count_limit_ok = classify_interactive_source("#\n" * 4_096)
utf8_boundary_line = "#" + "é" * 2_047 + "\n"
byte_limit_ok = classify_interactive_source(utf8_boundary_line * 16)

limit_failures = 0
for candidate in (
    "x" * 4_097,
    "#\n" * 4_097,
    utf8_boundary_line * 17,
    "#" + "a" * 4_094 + "\f" + "b" * 4_095,
):
    try:
        classify_interactive_source(candidate)
    except InteractiveSourceLimitError:
        limit_failures += 1

assert (
    complete == (InteractiveSourceStatus.COMPLETE,) * 3
    and incomplete == (InteractiveSourceStatus.INCOMPLETE,) * 4
    and invalid is InteractiveSourceStatus.INVALID
    and not_executed is InteractiveSourceStatus.COMPLETE
    and warning_case is InteractiveSourceStatus.COMPLETE
    and warning_state_restored
    and line_limit_ok in InteractiveSourceStatus
    and count_limit_ok in InteractiveSourceStatus
    and byte_limit_ok in InteractiveSourceStatus
    and limit_failures == 4
)
```

## Trade-offs and Limitations

The result describes CPython's current `single` interactive grammar only. It
does not expose syntax diagnostics, retain bytecode, evaluate names, import
modules, or execute the source. Compiler work and memory use are
implementation-dependent; the text, byte, line, and per-line limits make the
accepted profile finite but do not turn compilation into a sandbox.

Warnings are captured and discarded so they do not reach diagnostic output,
and the prior warning filter and hook are restored on context exit. On builds
where `sys.flags.context_aware_warnings` is false, that temporary state is
process-global and is not isolated from concurrent warning operations. Run
this helper in a serialized compiler lane in such environments. Re-test the
classification when changing Python versions because grammar decisions can
change.

## Related Snippets

<!-- catalog:related:start -->
- [Audit Bounded Python Source for Tab-Width-Independent Indentation](audit-bounded-python-source-for-tab-width-independent-indentation.md)
- [Compare Bounded Python Expressions Structurally without Execution](compare-bounded-python-expressions-structurally-without-execution.md)
- [Index Bounded Python Scope Bindings with symtable without Execution](index-bounded-python-scope-bindings-with-symtable-without-execution.md)
<!-- catalog:related:end -->
