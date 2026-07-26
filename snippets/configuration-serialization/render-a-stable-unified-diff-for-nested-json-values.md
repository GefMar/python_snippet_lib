---
title: "Render a Stable Unified Diff for Nested JSON Values"
snippet_type: recipe
use_cases:
  - serialization
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - fingerprint-a-set-like-json-array-deterministically.md
  - get-nested-values-with-a-validated-dot-path.md
  - ../data-processing/project-nested-records-with-explicit-field-paths.md
---

# Render a Stable Unified Diff for Nested JSON Values

## Idea and Problem

Flatten nested JSON into deterministic, type-preserving path lines before rendering a compact unified diff for human review.

Every container receives an `object` or `array` marker, and every scalar uses
canonical JSON text. Paths follow JSON Pointer token escaping, mapping keys are
sorted, and arrays keep numeric indexes. ASCII-escaped JSON strings keep every
path and scalar on one physical output line, even when source text contains
Unicode line separators.

## When to Use

Use this renderer in tests, review reports, or diagnostics where an ordinary
pretty-printed JSON diff produces noisy indentation changes. It works best for
bounded documents whose arrays are ordered. Use JSON Patch or a domain-aware
comparison when another program must apply the result or align entities by
identity.

## Implementation

```python
import difflib
import json
import math


def _pointer(tokens: tuple[str, ...]) -> str:
    return "".join(
        "/" + token.replace("~", "~0").replace("/", "~1")
        for token in tokens
    )


def _path_text(tokens: tuple[str, ...]) -> str:
    return json.dumps(_pointer(tokens), ensure_ascii=True)


def flatten_json_lines(value: object) -> tuple[str, ...]:
    lines: list[str] = []
    active: set[int] = set()

    def visit(node: object, path: tuple[str, ...]) -> None:
        path_text = _path_text(path)
        if node is None or type(node) in (bool, int, str):
            encoded = json.dumps(node, ensure_ascii=True, separators=(",", ":"))
            lines.append(f"{path_text}\tvalue\t{encoded}")
            return
        if type(node) is float:
            if not math.isfinite(node):
                raise ValueError("JSON numbers must be finite")
            encoded = json.dumps(node, allow_nan=False, separators=(",", ":"))
            lines.append(f"{path_text}\tvalue\t{encoded}")
            return
        if type(node) is dict:
            identity = id(node)
            if identity in active:
                raise ValueError("cyclic mappings are not valid JSON")
            if any(type(key) is not str for key in node):
                raise TypeError("JSON object keys must be strings")
            lines.append(f"{path_text}\tobject")
            active.add(identity)
            try:
                for key in sorted(node):
                    visit(node[key], path + (key,))
            finally:
                active.remove(identity)
            return
        if type(node) is list:
            identity = id(node)
            if identity in active:
                raise ValueError("cyclic lists are not valid JSON")
            lines.append(f"{path_text}\tarray")
            active.add(identity)
            try:
                for index, item in enumerate(node):
                    visit(item, path + (str(index),))
            finally:
                active.remove(identity)
            return
        raise TypeError(f"unsupported JSON value: {type(node).__name__}")

    visit(value, ())
    return tuple(lines)


def render_json_unified_diff(
    before: object,
    after: object,
    *,
    before_name: str = "before.json",
    after_name: str = "after.json",
    context_lines: int = 3,
    max_lines: int = 5_000,
) -> tuple[str, ...]:
    for label_name, label in (("before_name", before_name), ("after_name", after_name)):
        if not isinstance(label, str) or not label:
            raise ValueError(f"{label_name} must be non-empty text")
        if not label.isprintable():
            raise ValueError(f"{label_name} must contain only printable text")
    if isinstance(context_lines, bool) or not isinstance(context_lines, int):
        raise TypeError("context_lines must be an integer")
    if context_lines < 0:
        raise ValueError("context_lines must be non-negative")
    if isinstance(max_lines, bool) or not isinstance(max_lines, int):
        raise TypeError("max_lines must be an integer")
    if max_lines <= 0:
        raise ValueError("max_lines must be positive")

    before_lines = flatten_json_lines(before)
    after_lines = flatten_json_lines(after)
    if len(before_lines) > max_lines or len(after_lines) > max_lines:
        raise ValueError("flattened JSON exceeds max_lines")

    return tuple(
        difflib.unified_diff(
            before_lines,
            after_lines,
            fromfile=before_name,
            tofile=after_name,
            n=context_lines,
            lineterm="",
        )
    )
```

## Example

```python
before = {
    "a/b": {"~key": []},
    "items": [1],
}
after = {
    "items": [1, 2],
    "a/b": {"~key": []},
}
diff = render_json_unified_diff(
    before,
    after,
    before_name="saved.json",
    after_name="current.json",
)
same_content = render_json_unified_diff(
    {"second": 2, "first": 1},
    {"first": 1, "second": 2},
)

assert (
    diff[0],
    diff[1],
    any(line == '+"/items/1"\tvalue\t2' for line in diff),
    '"/a~1b/~0key"\tarray' in flatten_json_lines(before),
    same_content,
) == (
    "--- saved.json",
    "+++ current.json",
    True,
    True,
    (),
)
```

## Trade-offs and Limitations

Flattening materializes both line sequences, and `difflib` can take quadratic
time in difficult cases even below `max_lines`. Container markers preserve
object-versus-array structure, but array insertions shift later numeric paths
and can create a larger display diff. The output is diagnostic text, not a
minimal edit script, JSON Patch, or reversible format. Python's JSON scalar
formatting defines this local representation, with non-ASCII text escaped for
line safety. Very deep input can hit the recursion limit, and the line cap is
not a defense for hostile structures with extreme depth or scalar size. Values
appear in the diff, so redact secrets before calling the renderer.

## Related Snippets

<!-- catalog:related:start -->
- [Fingerprint a Set-Like JSON Array Deterministically](fingerprint-a-set-like-json-array-deterministically.md)
- [Get Nested Values with a Validated Dot Path](get-nested-values-with-a-validated-dot-path.md)
- [Project Nested Records with Explicit Field Paths](../data-processing/project-nested-records-with-explicit-field-paths.md)
<!-- catalog:related:end -->
