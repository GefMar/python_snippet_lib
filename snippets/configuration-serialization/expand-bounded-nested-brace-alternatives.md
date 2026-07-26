---
title: "Expand Bounded Nested Brace Alternatives"
snippet_type: algorithm
use_cases:
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - get-nested-values-with-a-validated-dot-path.md
  - parse-compact-durations-into-timedelta.md
---

# Expand Bounded Nested Brace Alternatives

## Idea and Problem

Expand a small brace-alternative grammar deterministically while bounding the number of materialized results.

Each brace group contains two or more comma-separated alternatives, and an
alternative may contain another group. A structural validation pass rejects
ambiguous input before expansion; a result limit then prevents combinatorial
growth from allocating an unbounded list.

## When to Use

Use this algorithm when trusted configuration needs compact, ordered choices
such as `node-{api,worker{1,2}}-{blue,green}`. Define both a nesting limit and
an output limit appropriate for the caller before accepting external input.
Use a dedicated shell-compatible library when ranges, quoting, or escaping are
required; this deliberately small grammar never executes anything.

## Implementation

```python
def _validate_braces(pattern: str, *, max_depth: int) -> None:
    depth = 0
    for character in pattern:
        if character == "\\":
            raise ValueError("escaping is not supported")
        if character == "{":
            depth += 1
            if depth > max_depth:
                raise ValueError("brace nesting exceeds max_depth")
        elif character == "}":
            if depth == 0:
                raise ValueError("unexpected closing brace")
            depth -= 1
    if depth:
        raise ValueError("unclosed brace")


def _first_group(pattern: str) -> tuple[int, int] | None:
    start: int | None = None
    depth = 0
    for index, character in enumerate(pattern):
        if character == "{":
            if depth == 0:
                start = index
            depth += 1
        elif character == "}":
            depth -= 1
            if depth == 0:
                assert start is not None
                return start, index
    return None


def _split_alternatives(body: str, *, max_alternatives: int) -> list[str]:
    parts: list[str] = []
    depth = 0
    start = 0
    for index, character in enumerate(body):
        if character == "{":
            depth += 1
        elif character == "}":
            depth -= 1
        elif character == "," and depth == 0:
            parts.append(body[start:index])
            if len(parts) >= max_alternatives:
                raise ValueError("brace expansion exceeds max_results")
            start = index + 1
    parts.append(body[start:])

    if len(parts) > max_alternatives:
        raise ValueError("brace expansion exceeds max_results")

    if len(parts) < 2:
        raise ValueError("a brace group requires at least two alternatives")
    if any(part == "" for part in parts):
        raise ValueError("empty brace alternatives are not supported")
    return parts


def expand_brace_alternatives(
    pattern: str,
    *,
    max_results: int = 256,
    max_depth: int = 16,
) -> list[str]:
    if not isinstance(pattern, str):
        raise TypeError("pattern must be text")
    for name, value in (("max_results", max_results), ("max_depth", max_depth)):
        if isinstance(value, bool) or not isinstance(value, int):
            raise TypeError(f"{name} must be an integer")
        if value <= 0:
            raise ValueError(f"{name} must be positive")

    _validate_braces(pattern, max_depth=max_depth)
    results: list[str] = []
    pending = [pattern]
    while pending:
        current = pending.pop()
        group = _first_group(current)
        if group is None:
            results.append(current)
            continue

        start, end = group
        prefix = current[:start]
        suffix = current[end + 1 :]
        alternatives = _split_alternatives(
            current[start + 1 : end],
            max_alternatives=max_results,
        )
        if len(results) + len(pending) + len(alternatives) > max_results:
            raise ValueError("brace expansion exceeds max_results")
        pending.extend(
            prefix + alternative + suffix
            for alternative in reversed(alternatives)
        )

    return results
```

## Example

```python
expanded = expand_brace_alternatives(
    "node-{api,worker{1,2}}-{blue,green}",
    max_results=6,
)

try:
    expand_brace_alternatives("{a,b}{c,d}", max_results=3)
except ValueError:
    limit_enforced = True
else:
    limit_enforced = False

try:
    expand_brace_alternatives("value-{single}")
except ValueError:
    singleton_rejected = True
else:
    singleton_rejected = False

assert (expanded, limit_enforced, singleton_rejected) == (
    [
        "node-api-blue",
        "node-api-green",
        "node-worker1-blue",
        "node-worker1-green",
        "node-worker2-blue",
        "node-worker2-green",
    ],
    True,
    True,
)
```

## Trade-offs and Limitations

This grammar supports literal text, commas, braces, and nesting only. It does
not implement shell ranges, escaping, quoting, environment expansion, or glob
matching, and it preserves duplicate alternatives rather than deduplicating
them. The function returns a complete list, so memory remains proportional to
the allowed result count and result lengths. A pending-branch lower bound
rejects an oversized product before those results are materialized, but callers
must still keep the input length and limits bounded before parsing untrusted
data.

## Related Snippets

<!-- catalog:related:start -->
- [Get Nested Values with a Validated Dot Path](get-nested-values-with-a-validated-dot-path.md)
- [Parse Compact Durations into timedelta](parse-compact-durations-into-timedelta.md)
<!-- catalog:related:end -->
