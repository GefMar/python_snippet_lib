---
title: "Parse a Bounded Explicit CLI Argument Tuple Without Reading sys.argv or Exiting"
snippet_type: recipe
use_cases:
  - configuration
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - tokenize-a-bounded-posix-style-argument-string-without-expansion.md
  - normalize-bounded-named-options-with-explicit-default-semantics.md
  - reject-unknown-options-with-conservative-typo-suggestions.md
---

# Parse a Bounded Explicit CLI Argument Tuple Without Reading sys.argv or Exiting

## Idea and Problem

Parse one explicit, size-bounded argument tuple against a fixed command schema without consulting process-global arguments, writing diagnostics, or terminating the interpreter.

`ArgumentParser` normally serves a process entry point, where printing usage
and raising `SystemExit` are appropriate. An embedded boundary needs different
failure semantics. This parser removes help and version actions, disables
abbreviations and suggestions, overrides both exit paths, and converts every
parse failure into one value-free domain exception.

## When to Use

Use this when a test harness, dispatcher, notebook, or larger application
already owns an exact tuple of arguments and needs one narrow reusable parser.
The fixed schema accepts only output format, result limit, and verbosity.

Use a normal command-line entry point when users need generated help, rich
diagnostics, subcommands, shell completion, or process exit codes. Tokenize a
command string separately; this function accepts already separated arguments
and performs no shell expansion, environment lookup, file loading, or command
execution.

## Implementation

```python
import argparse
from contextlib import redirect_stderr, redirect_stdout
from dataclasses import dataclass
from io import StringIO

_MAX_ARGUMENTS = 16
_MAX_ARGUMENT_BYTES = 256
_MAX_TOTAL_ARGUMENT_BYTES = 2_048
_KNOWN_OPTIONS = frozenset(("--format", "--limit", "--verbose"))


class CliArgumentsError(ValueError):
    """Raised when arguments do not match the embedded command profile."""


@dataclass(frozen=True, slots=True)
class ReportOptions:
    output_format: str
    limit: int
    verbose: bool


class _NonExitingParser(argparse.ArgumentParser):
    def error(self, message: str) -> None:
        raise CliArgumentsError(
            "arguments do not match the supported command schema"
        )

    def exit(self, status: int = 0, message: str | None = None) -> None:
        raise CliArgumentsError("the parser exit path is disabled")


def _parse_limit(value: str) -> int:
    if not value.isascii() or not value.isdecimal():
        raise argparse.ArgumentTypeError("limit must be an ASCII decimal integer")
    limit = int(value, 10)
    if not 1 <= limit <= 1_000:
        raise argparse.ArgumentTypeError("limit is outside the supported range")
    return limit


def parse_report_argv(argv: tuple[str, ...]) -> ReportOptions:
    """Parse explicit arguments without reading sys.argv or exiting."""
    if type(argv) is not tuple:
        raise TypeError("argv must be an exact tuple")
    if len(argv) > _MAX_ARGUMENTS:
        raise CliArgumentsError("argv exceeds the argument-count limit")

    total_argument_bytes = 0
    seen_options: set[str] = set()
    for token in argv:
        if type(token) is not str:
            raise TypeError("each argv item must be an exact string")
        try:
            token_bytes = len(token.encode("utf-8"))
        except UnicodeEncodeError:
            raise CliArgumentsError(
                "each argv item must be UTF-8 encodable"
            ) from None
        if token_bytes > _MAX_ARGUMENT_BYTES:
            raise CliArgumentsError("an argv item exceeds the byte limit")
        total_argument_bytes += token_bytes
        if total_argument_bytes > _MAX_TOTAL_ARGUMENT_BYTES:
            raise CliArgumentsError("argv exceeds the aggregate byte limit")
        if any(ord(character) < 0x20 or ord(character) == 0x7F for character in token):
            raise CliArgumentsError("argv items must not contain control characters")
        if token == "--":
            raise CliArgumentsError("the end-of-options marker is not supported")

        option = token.partition("=")[0]
        if option in _KNOWN_OPTIONS:
            if option in seen_options:
                raise CliArgumentsError("an option was provided more than once")
            seen_options.add(option)

    parser = _NonExitingParser(
        prog="report",
        add_help=False,
        allow_abbrev=False,
        exit_on_error=False,
        suggest_on_error=False,
        color=False,
    )
    parser.add_argument(
        "--format",
        dest="output_format",
        choices=("json", "text"),
        default="text",
    )
    parser.add_argument("--limit", type=_parse_limit, default=100)
    parser.add_argument("--verbose", action="store_true")

    try:
        namespace = parser.parse_args(argv)
    except argparse.ArgumentError:
        raise CliArgumentsError(
            "arguments do not match the supported command schema"
        ) from None

    return ReportOptions(
        output_format=namespace.output_format,
        limit=namespace.limit,
        verbose=namespace.verbose,
    )
```

## Example

```python
captured_stdout = StringIO()
captured_stderr = StringIO()

with redirect_stdout(captured_stdout), redirect_stderr(captured_stderr):
    parsed = parse_report_argv(
        ("--format=json", "--limit", "3", "--verbose")
    )
    defaults = parse_report_argv(())

    rejected = 0
    for invalid in (
        ("--for=json",),
        ("--format", "yaml"),
        ("--limit", "0"),
        ("--limit=3", "--limit", "4"),
        ("--verbose", "--verbose=true"),
        ("--",),
        ("-h",),
    ):
        try:
            parse_report_argv(invalid)
        except CliArgumentsError:
            rejected += 1

assert (
    parsed == ReportOptions("json", 3, True)
    and defaults == ReportOptions("text", 100, False)
    and rejected == 7
    and captured_stdout.getvalue() == ""
    and captured_stderr.getvalue() == ""
)
```

## Trade-offs and Limitations

Validation and parsing are linear in the bounded argument count and inspected
UTF-8 bytes. A fresh parser is built for each call so no mutable parser state
is shared. Duplicate recognized options are rejected before `argparse` can
apply its normal last-value-wins behavior, including mixed `--name value` and
`--name=value` forms.

`exit_on_error=False` alone does not cover every parser error. Overriding
`error()` handles usage failures such as unknown options, while overriding
`exit()` protects the remaining parser exit path. Help and version actions are
not installed because their intended behavior is to exit successfully.

All failures deliberately discard detailed messages that could echo argument
values. The profile has no positional arguments, abbreviations, suggestions,
colors, response files, subcommands, shell semantics, or localization.
Parsing establishes syntax only; it does not authorize the requested work or
make later file, network, or output operations safe.

## Related Snippets

<!-- catalog:related:start -->
- [Tokenize a Bounded POSIX-Style Argument String Without Expansion](tokenize-a-bounded-posix-style-argument-string-without-expansion.md)
- [Normalize Bounded Named Options with Explicit Default Semantics](normalize-bounded-named-options-with-explicit-default-semantics.md)
- [Reject Unknown Options with Conservative Typo Suggestions](reject-unknown-options-with-conservative-typo-suggestions.md)
<!-- catalog:related:end -->
