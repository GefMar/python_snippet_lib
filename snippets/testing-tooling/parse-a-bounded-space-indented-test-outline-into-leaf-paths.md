---
title: "Parse a Bounded Space-Indented Test Outline into Leaf Paths"
snippet_type: algorithm
use_cases:
  - parsing
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/parse-a-bounded-nested-bracket-tree.md
  - ../data-processing/parse-pipe-delimited-tables-with-continuation-rows.md
  - ../data-processing/collect-expected-parse-failures-without-stopping-a-batch.md
---

# Parse a Bounded Space-Indented Test Outline into Leaf Paths

## Idea and Problem

Turn one strict space-indented test listing into immutable leaf paths without silently repairing malformed hierarchy.

Human-readable test runners sometimes expose only an outline of suites and
tests. Tracking the current ancestor stack is enough to recover leaf paths, but
the parser must reject partial indentation, depth jumps, controls, and duplicate
paths that would otherwise create an ambiguous execution target.

## When to Use

Use this adapter for a trusted command whose documented listing format uses a
fixed number of ASCII spaces per level and whose output must be converted into
individual test selectors. Preserve tuple components until the target runner
defines an escaping or joining rule. Prefer JSON, XML, or another machine
format whenever the runner provides one.

## Implementation

```python
_MAX_OUTLINE_CHARACTERS = 65_536
_MAX_OUTLINE_LINES = 1000
_MAX_OUTLINE_DEPTH = 32
_MAX_NODE_NAME_CHARACTERS = 128


class IndentedOutlineError(ValueError):
    pass


def parse_test_leaf_paths(
    text: str,
    *,
    indent_width: int = 2,
) -> tuple[tuple[str, ...], ...]:
    if not isinstance(text, str):
        raise TypeError("text must be a string")
    if len(text) > _MAX_OUTLINE_CHARACTERS:
        raise ValueError("text exceeds the supported length")
    if isinstance(indent_width, bool) or not isinstance(indent_width, int):
        raise TypeError("indent_width must be an integer")
    if not 1 <= indent_width <= 8:
        raise ValueError("indent_width is outside the supported range")
    if "\r" in text:
        raise IndentedOutlineError("carriage returns are not supported")
    if any(
        character != "\n" and not character.isprintable()
        for character in text
    ):
        raise IndentedOutlineError("text contains a control character")

    lines = text.split("\n")
    if len(lines) > _MAX_OUTLINE_LINES:
        raise ValueError("text exceeds the supported line count")

    records: list[tuple[int, tuple[str, ...]]] = []
    stack: list[str] = []
    seen_paths: set[tuple[str, ...]] = set()
    previous_level = 0

    for line_number, line in enumerate(lines, start=1):
        if line == "":
            continue
        if line.endswith(" "):
            raise IndentedOutlineError(
                f"line {line_number} has trailing whitespace"
            )

        indentation = len(line) - len(line.lstrip(" "))
        if indentation % indent_width:
            raise IndentedOutlineError(
                f"line {line_number} has partial indentation"
            )
        level = indentation // indent_width
        if level >= _MAX_OUTLINE_DEPTH:
            raise IndentedOutlineError(
                f"line {line_number} exceeds the supported depth"
            )
        if not records and level != 0:
            raise IndentedOutlineError("the first node must be at root level")
        if records and level > previous_level + 1:
            raise IndentedOutlineError(
                f"line {line_number} skips an indentation level"
            )

        name = line[indentation:]
        if not 1 <= len(name) <= _MAX_NODE_NAME_CHARACTERS:
            raise IndentedOutlineError(
                f"line {line_number} has an invalid node name length"
            )

        stack[level:] = [name]
        path = tuple(stack)
        if path in seen_paths:
            raise IndentedOutlineError(
                f"line {line_number} repeats an existing path"
            )
        seen_paths.add(path)
        records.append((level, path))
        previous_level = level

    leaves: list[tuple[str, ...]] = []
    for index, (level, path) in enumerate(records):
        is_leaf = (
            index + 1 == len(records)
            or records[index + 1][0] <= level
        )
        if is_leaf:
            leaves.append(path)
    return tuple(leaves)
```

## Example

```python
listing = """suite-a
  test-one
  nested
    test-two

suite-b
  test-three
"""

leaf_paths = parse_test_leaf_paths(listing, indent_width=2)

assert leaf_paths == (
    ("suite-a", "test-one"),
    ("suite-a", "nested", "test-two"),
    ("suite-b", "test-three"),
)
```

## Trade-offs and Limitations

This is a strict adapter for one small grammar, not a general CLI-output
recovery parser. It accepts empty lines but rejects whitespace-only lines,
trailing spaces, tabs, carriage returns, other non-printable characters,
misaligned indentation, and a depth increase greater than one. Node names are
kept literally; source-specific decorations are not removed.

The parser materializes bounded records before identifying leaves, using
linear time and memory in the number of nodes plus the stored path components.
It preserves input order and allows identical names under different parents,
but rejects a repeated full path. It does not join components into a string,
escape runner-specific separators, invoke tests, or validate whether a leaf
still exists in the target binary.

## Related Snippets

<!-- catalog:related:start -->
- [Parse a Bounded Nested Bracket Tree](../configuration-serialization/parse-a-bounded-nested-bracket-tree.md)
- [Parse Pipe-Delimited Tables with Continuation Rows](../data-processing/parse-pipe-delimited-tables-with-continuation-rows.md)
- [Collect Expected Parse Failures Without Stopping a Batch](../data-processing/collect-expected-parse-failures-without-stopping-a-batch.md)
<!-- catalog:related:end -->
