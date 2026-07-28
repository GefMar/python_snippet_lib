---
title: "Parse a Bounded Debugger Function Listing into Canonical Source Locations"
snippet_type: testing-technique
use_cases:
  - parsing
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - scan-bounded-macro-declarations-into-a-canonical-event-index.md
  - extract-bounded-native-test-failure-highlights.md
  - ../concurrency-lifecycle/stream-bounded-stdout-and-stderr-lines-from-a-posix-process.md
---

# Parse a Bounded Debugger Function Listing into Canonical Source Locations

## Idea and Problem

Parse a narrow, size-bounded GNU GDB function-listing snapshot into deterministic source locations without reading binaries or source files.

`info functions -n` can group line-annotated functions under `File ...:`
headers when suitable debug information is present. A test or build step may
need only the unique file-and-line locations, not declarations or symbol
values. A stateful parser can retain the active file header, discard declaration
text, and map machine-specific absolute prefixes into neutral logical paths.

## When to Use

Use this parser after a separate trusted process runner has captured the
relevant file-section subset of GDB output. The accepted subset contains only
blank lines, `File <POSIX path>:` headers, and positive-line records such as
`27: void initialize_clock();`. Supply prefix mappings for every allowed
absolute build root; already-relative canonical paths need no mapping.

This is useful for deterministic auditing or test indexing when the exact GDB
version, language printer, compiler, and debug-information settings are pinned.
Do not treat it as a general parser for every GDB command or output version.

## Implementation

```python
import re
from dataclasses import dataclass


_MAX_TEXT_BYTES = 1 << 20
_MAX_LINES = 4_096
_MAX_LINE_CHARS = 2_048
_MAX_RECORDS = 512
_MAX_MAPPINGS = 8
_MAX_PATH_BYTES = 1_024
_MAX_PATH_PARTS = 64
_FILE_HEADER = re.compile(r"File ([^\n]+):", re.ASCII)
_FUNCTION_LINE = re.compile(r"([1-9][0-9]{0,8}):[ \t]+\S.*", re.ASCII)


@dataclass(frozen=True, slots=True)
class SourcePrefix:
    absolute_root: str
    logical_root: str


@dataclass(frozen=True, order=True, slots=True)
class SourceLocation:
    logical_path: str
    line: int


def _path_parts(raw: object, *, absolute: bool, field: str) -> tuple[str, ...]:
    if type(raw) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not raw or len(raw.encode("utf-8")) > _MAX_PATH_BYTES:
        raise ValueError(f"{field} is empty or too large")
    if "\x00" in raw or "\\" in raw or "//" in raw:
        raise ValueError(f"{field} is not canonical POSIX text")
    if raw.endswith("/"):
        raise ValueError(f"{field} must not end with a slash")
    if raw.startswith("/") != absolute:
        raise ValueError(f"{field} has the wrong path kind")

    payload = raw[1:] if absolute else raw
    parts = tuple(payload.split("/"))
    if not parts or len(parts) > _MAX_PATH_PARTS:
        raise ValueError(f"{field} has an invalid component count")
    if any(part in ("", ".", "..") for part in parts):
        raise ValueError(f"{field} contains a forbidden component")
    return parts


def _prefixes(value: object) -> tuple[tuple[tuple[str, ...], tuple[str, ...]], ...]:
    if type(value) is not tuple:
        raise TypeError("prefixes must be an exact tuple")
    if len(value) > _MAX_MAPPINGS:
        raise ValueError("prefix mapping limit exceeded")

    result: list[tuple[tuple[str, ...], tuple[str, ...]]] = []
    for mapping in value:
        if type(mapping) is not SourcePrefix:
            raise TypeError("prefixes must contain exact SourcePrefix records")
        source = _path_parts(
            mapping.absolute_root,
            absolute=True,
            field="absolute_root",
        )
        logical = _path_parts(
            mapping.logical_root,
            absolute=False,
            field="logical_root",
        )
        for previous, _ in result:
            shared = min(len(source), len(previous))
            if source[:shared] == previous[:shared]:
                raise ValueError("absolute prefix mappings overlap")
        result.append((source, logical))
    return tuple(result)


def _logical_path(
    raw_path: str,
    prefixes: tuple[tuple[tuple[str, ...], tuple[str, ...]], ...],
) -> str:
    if raw_path.startswith("/"):
        parts = _path_parts(raw_path, absolute=True, field="listed path")
        matches = [
            (source, logical)
            for source, logical in prefixes
            if len(parts) > len(source) and parts[: len(source)] == source
        ]
        if len(matches) != 1:
            raise ValueError("absolute listed path has no unique prefix mapping")
        source, logical = matches[0]
        output = (*logical, *parts[len(source) :])
    else:
        output = _path_parts(raw_path, absolute=False, field="listed path")

    rendered = "/".join(output)
    if len(output) > _MAX_PATH_PARTS or len(rendered.encode("utf-8")) > _MAX_PATH_BYTES:
        raise ValueError("canonical logical path exceeds its limit")
    return rendered


def parse_debugger_function_locations(
    text: str,
    prefix_mappings: tuple[SourcePrefix, ...] = (),
) -> tuple[SourceLocation, ...]:
    if type(text) is not str:
        raise TypeError("text must be an exact string")
    if "\r" in text or "\x00" in text:
        raise ValueError("text contains a forbidden control character")
    if len(text.encode("utf-8")) > _MAX_TEXT_BYTES:
        raise ValueError("text limit exceeded")

    lines = text.split("\n")
    if lines[-1] == "":
        lines.pop()
    if len(lines) > _MAX_LINES:
        raise ValueError("line-count limit exceeded")
    if any(len(line) > _MAX_LINE_CHARS for line in lines):
        raise ValueError("line-length limit exceeded")

    prefixes = _prefixes(prefix_mappings)
    active_path: str | None = None
    active_has_record = False
    locations: set[SourceLocation] = set()

    for line in lines:
        if not line:
            continue
        if header := _FILE_HEADER.fullmatch(line):
            if active_path is not None and not active_has_record:
                raise ValueError("file header has no function record")
            active_path = _logical_path(header.group(1), prefixes)
            active_has_record = False
            continue
        if record := _FUNCTION_LINE.fullmatch(line):
            if active_path is None:
                raise ValueError("function record appears before a file header")
            locations.add(SourceLocation(active_path, int(record.group(1))))
            if len(locations) > _MAX_RECORDS:
                raise ValueError("location limit exceeded")
            active_has_record = True
            continue
        raise ValueError("line is outside the supported listing subset")

    if active_path is not None and not active_has_record:
        raise ValueError("final file header has no function record")
    return tuple(sorted(locations))
```

## Example

```python
listing = """File /compile/root/lib/clock.c:
27: void initialize_clock();
27: void initialize_clock_alias();

File modules/math.c:
8: double clamp(double, double, double);
"""

locations = parse_debugger_function_locations(
    listing,
    (SourcePrefix("/compile/root", "workspace"),),
)

assert locations == (
    SourceLocation("modules/math.c", 8),
    SourceLocation("workspace/lib/clock.c", 27),
)
```

## Trade-offs and Limitations

Parsing is linear in at most 1 MiB and 4,096 lines; sorting at most 512 unique
records costs `O(r log r)` time. The parser deliberately rejects GDB preambles,
address-only symbols, unannotated functions, alternate file headers, wrapped
declarations, carriage returns, overlapping prefix maps, and root-only absolute
matches. Deduplication by file and line also hides multiple functions reported
at the same location.

The caller is responsible for invoking a pinned debugger safely, bounding its
runtime and captured output, and extracting this narrow subset. Results depend
on compiler and linker behavior, available debug information, language-specific
printing, and GDB version. The function launches no process, opens no binary or
source file, imports no target code, reads no symbol value, and writes no report.

## Related Snippets

<!-- catalog:related:start -->
- [Scan Bounded Macro Declarations into a Canonical Event Index](scan-bounded-macro-declarations-into-a-canonical-event-index.md)
- [Extract Bounded Native-Test Failure Highlights](extract-bounded-native-test-failure-highlights.md)
- [Stream Bounded stdout and stderr Lines from a POSIX Process](../concurrency-lifecycle/stream-bounded-stdout-and-stderr-lines-from-a-posix-process.md)
<!-- catalog:related:end -->
