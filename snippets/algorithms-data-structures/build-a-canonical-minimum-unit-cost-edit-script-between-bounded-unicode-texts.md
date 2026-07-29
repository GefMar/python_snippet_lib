---
title: "Build a Canonical Minimum Unit-Cost Edit Script Between Bounded Unicode Texts"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - rank-hierarchy-paths-with-bounded-weighted-edit-distance.md
  - find-a-longest-common-integer-subsequence-with-earliest-index-pair-ties.md
  - ../configuration-serialization/render-a-stable-unified-diff-for-nested-json-values.md
---

# Build a Canonical Minimum Unit-Cost Edit Script Between Bounded Unicode Texts

## Idea and Problem

Reconstruct one deterministic minimum Levenshtein edit script instead of returning only an edit distance.

Each `equal`, `replace`, `delete`, or `insert` step records source and target
indices before it consumes a code point. Unit-cost replacement, deletion, and
insertion contribute one to the distance; an equal step contributes zero.

A suffix-distance table makes every optimal next transition visible during a
forward walk. Equal code points are kept first. At a mismatch, the first
optimal transition in `replace`, `delete`, `insert` order defines the
canonical result.

## When to Use

Use this algorithm when bounded text must be aligned reproducibly and callers
need replayable operations, not just a similarity score. The fixed tie rule is
useful in tests, review tools, and deterministic transformations where several
minimum scripts may exist.

The units are Python Unicode code points. Normalize or segment text before
calling only when that policy is explicitly part of the surrounding system.

## Implementation

```python
from dataclasses import dataclass
from enum import StrEnum

_MAX_CODE_POINTS = 512
_MAX_UTF8_BYTES = 2_048
_MAX_TABLE_CELLS = 263_169


class EditOperation(StrEnum):
    EQUAL = "equal"
    REPLACE = "replace"
    DELETE = "delete"
    INSERT = "insert"


@dataclass(frozen=True, slots=True)
class EditStep:
    operation: EditOperation
    source_index: int
    target_index: int
    source_character: str | None
    target_character: str | None


@dataclass(frozen=True, slots=True)
class EditScript:
    distance: int
    steps: tuple[EditStep, ...]


def _validate_text(name: str, value: object) -> str:
    if type(value) is not str:
        raise TypeError(f"{name} must be an exact string")
    if len(value) > _MAX_CODE_POINTS:
        raise ValueError(f"{name} exceeds the code-point limit")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError as error:
        raise ValueError(f"{name} contains a surrogate code point") from error
    if len(encoded) > _MAX_UTF8_BYTES:
        raise ValueError(f"{name} exceeds the UTF-8 byte limit")
    return value


def minimum_unit_cost_edit_script(source: str, target: str) -> EditScript:
    """Return one canonical minimum edit script from source to target."""
    source = _validate_text("source", source)
    target = _validate_text("target", target)

    source_length = len(source)
    target_length = len(target)
    cell_count = (source_length + 1) * (target_length + 1)
    if cell_count > _MAX_TABLE_CELLS:
        raise ValueError("edit-distance table exceeds the supported size")

    suffix_distance = [
        [0] * (target_length + 1)
        for _ in range(source_length + 1)
    ]
    for source_index in range(source_length, -1, -1):
        suffix_distance[source_index][target_length] = source_length - source_index
    for target_index in range(target_length, -1, -1):
        suffix_distance[source_length][target_index] = target_length - target_index

    for source_index in range(source_length - 1, -1, -1):
        for target_index in range(target_length - 1, -1, -1):
            if source[source_index] == target[target_index]:
                suffix_distance[source_index][target_index] = suffix_distance[
                    source_index + 1
                ][target_index + 1]
            else:
                suffix_distance[source_index][target_index] = 1 + min(
                    suffix_distance[source_index + 1][target_index + 1],
                    suffix_distance[source_index + 1][target_index],
                    suffix_distance[source_index][target_index + 1],
                )

    steps: list[EditStep] = []
    source_index = 0
    target_index = 0
    while source_index < source_length or target_index < target_length:
        remaining = suffix_distance[source_index][target_index]

        if (
            source_index < source_length
            and target_index < target_length
            and source[source_index] == target[target_index]
            and remaining
            == suffix_distance[source_index + 1][target_index + 1]
        ):
            steps.append(
                EditStep(
                    EditOperation.EQUAL,
                    source_index,
                    target_index,
                    source[source_index],
                    target[target_index],
                )
            )
            source_index += 1
            target_index += 1
            continue

        if (
            source_index < source_length
            and target_index < target_length
            and remaining
            == 1 + suffix_distance[source_index + 1][target_index + 1]
        ):
            steps.append(
                EditStep(
                    EditOperation.REPLACE,
                    source_index,
                    target_index,
                    source[source_index],
                    target[target_index],
                )
            )
            source_index += 1
            target_index += 1
            continue

        if (
            source_index < source_length
            and remaining == 1 + suffix_distance[source_index + 1][target_index]
        ):
            steps.append(
                EditStep(
                    EditOperation.DELETE,
                    source_index,
                    target_index,
                    source[source_index],
                    None,
                )
            )
            source_index += 1
            continue

        if target_index < target_length:
            steps.append(
                EditStep(
                    EditOperation.INSERT,
                    source_index,
                    target_index,
                    None,
                    target[target_index],
                )
            )
            target_index += 1
            continue

        raise RuntimeError("edit script reconstruction reached an invalid state")

    return EditScript(suffix_distance[0][0], tuple(steps))
```

## Example

```python
def replay(source: str, script: EditScript) -> str:
    source_cursor = 0
    target_cursor = 0
    output: list[str] = []

    for step in script.steps:
        assert (step.source_index, step.target_index) == (
            source_cursor,
            target_cursor,
        )
        if step.source_character is not None:
            assert source[source_cursor] == step.source_character
            source_cursor += 1
        if step.target_character is not None:
            output.append(step.target_character)
            target_cursor += 1

    assert source_cursor == len(source)
    return "".join(output)


kitten_script = minimum_unit_cost_edit_script("kitten", "sitting")
assert kitten_script.distance == 3
assert replay("kitten", kitten_script) == "sitting"
assert sum(
    step.operation is not EditOperation.EQUAL
    for step in kitten_script.steps
) == kitten_script.distance

tie_script = minimum_unit_cost_edit_script("ab", "ba")
assert tuple(step.operation for step in tie_script.steps) == (
    EditOperation.REPLACE,
    EditOperation.REPLACE,
)

unicode_script = minimum_unit_cost_edit_script("café ☕", "cafe ☕")
assert replay("café ☕", unicode_script) == "cafe ☕"
```

## Trade-offs and Limitations

For source length `n` and target length `m`, the algorithm uses `O(nm)` time
and table state, plus at most `n + m` returned steps. At the configured limits,
the table contains at most 263,169 Python integer references; their actual
memory footprint is larger than the mathematical cell count.

The tie rule chooses one minimum sequence. It does not minimize edit runs,
match a line-oriented diff utility, or enumerate every optimum. A two-row
distance calculation uses less memory but cannot reconstruct this same
canonical script without additional work.

No normalization, case folding, grapheme segmentation, transposition,
operation weighting, affine-gap scoring, or line-hunk rendering is performed.
Visually identical text can therefore have a nonzero code-point distance.

## Related Snippets

<!-- catalog:related:start -->
- [Rank Hierarchy Paths with Bounded Weighted Edit Distance](rank-hierarchy-paths-with-bounded-weighted-edit-distance.md)
- [Find a Longest Common Integer Subsequence with Earliest Index-Pair Ties](find-a-longest-common-integer-subsequence-with-earliest-index-pair-ties.md)
- [Render a Stable Unified Diff for Nested JSON Values](../configuration-serialization/render-a-stable-unified-diff-for-nested-json-values.md)
<!-- catalog:related:end -->
