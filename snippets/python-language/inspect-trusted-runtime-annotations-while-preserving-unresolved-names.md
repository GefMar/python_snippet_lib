---
title: "Inspect Trusted Runtime Annotations while Preserving Unresolved Names"
snippet_type: recipe
use_cases:
  - automation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../testing-tooling/extract-a-bounded-static-python-annotation-index-without-evaluation.md
  - check-a-bounded-abstract-call-shape-against-a-signature-without-invocation.md
---

# Inspect Trusted Runtime Annotations while Preserving Unresolved Names

## Idea and Problem

Inspect one trusted Python function's runtime annotations while preserving simple unresolved dotted names and never returning arbitrary annotation objects.

Python 3.14's `annotationlib.Format.FORWARDREF` allows an annotation thunk to
produce top-level `ForwardRef` objects when a name cannot be resolved. After
introspection, this helper admits only objects matched by identity against an
explicit allowlist or exact `ForwardRef` values containing a bounded ASCII
dotted name. It returns a sorted tuple of frozen records containing safe text
labels rather than the original values.

## When to Use

Use this recipe at a trusted runtime extension boundary where one exact Python
function is already authorized and a small adapter needs its direct parameter
and return annotations. The identity allowlist makes recognized runtime
objects explicit, while unresolved names can be handed to a separate registry
or diagnostic layer without evaluation.

**Runtime annotation introspection may execute arbitrary Python code.** The
annotation thunk runs before this helper can inspect or cap its result, so the
operation is not pre-bounded and must never be applied to untrusted functions.
Use AST-based static analysis for untrusted source, non-executing audits, or a
hard resource boundary.

## Implementation

```python
import re
from annotationlib import Format, ForwardRef, get_annotations
from dataclasses import dataclass
from types import FunctionType

_MAX_RUNTIME_ANNOTATIONS = 64
_MAX_ALLOWED_ANNOTATIONS = 64
_MAX_ANNOTATION_NAME_CHARACTERS = 128
_MAX_LABEL_CHARACTERS = 256
_SIMPLE_NAME = re.compile(r"[A-Za-z_][A-Za-z0-9_]*\Z", re.ASCII)
_DOTTED_NAME = re.compile(
    r"[A-Za-z_][A-Za-z0-9_]*(?:\.[A-Za-z_][A-Za-z0-9_]*)*\Z",
    re.ASCII,
)


class RuntimeAnnotationProfileError(ValueError):
    """Raised when runtime annotations fall outside the closed profile."""


@dataclass(frozen=True, slots=True)
class AllowedAnnotation:
    value: object
    label: str


@dataclass(frozen=True, slots=True)
class RuntimeAnnotationLabel:
    name: str
    label: str
    unresolved: bool


def _validate_dotted_label(label: object, *, field: str) -> str:
    if type(label) is not str:
        raise TypeError(f"{field} must be an exact string")
    if not 1 <= len(label) <= _MAX_LABEL_CHARACTERS:
        raise RuntimeAnnotationProfileError(
            f"{field} is outside the character limit",
        )
    if _DOTTED_NAME.fullmatch(label) is None:
        raise RuntimeAnnotationProfileError(
            f"{field} must be a simple ASCII dotted name",
        )
    return label


def _validate_allowlist(
    allowed: tuple[AllowedAnnotation, ...],
) -> tuple[AllowedAnnotation, ...]:
    if type(allowed) is not tuple:
        raise TypeError("allowed must be an exact tuple")
    if len(allowed) > _MAX_ALLOWED_ANNOTATIONS:
        raise RuntimeAnnotationProfileError("allowed has more than 64 entries")

    labels: set[str] = set()
    checked: list[AllowedAnnotation] = []
    for entry in allowed:
        if type(entry) is not AllowedAnnotation:
            raise TypeError("allowed entries must be exact AllowedAnnotation values")
        label = _validate_dotted_label(entry.label, field="allowlist label")
        if label in labels:
            raise RuntimeAnnotationProfileError("allowlist labels must be unique")
        if any(entry.value is previous.value for previous in checked):
            raise RuntimeAnnotationProfileError(
                "allowlist object identities must be unique",
            )
        labels.add(label)
        checked.append(entry)
    return tuple(checked)


def inspect_trusted_runtime_annotations(
    function: FunctionType,
    allowed: tuple[AllowedAnnotation, ...],
) -> tuple[RuntimeAnnotationLabel, ...]:
    """Return safe labels for one trusted function's direct annotations."""
    if type(function) is not FunctionType:
        raise TypeError("function must be an exact Python function")
    checked_allowed = _validate_allowlist(allowed)

    # This call can execute the trusted function's annotation thunk. Its work
    # and allocation cannot be bounded by checks that run after it returns.
    annotations = get_annotations(
        function,
        format=Format.FORWARDREF,
        eval_str=False,
    )
    if type(annotations) is not dict:
        raise TypeError("annotationlib must return an exact dictionary")
    if len(annotations) > _MAX_RUNTIME_ANNOTATIONS:
        raise RuntimeAnnotationProfileError(
            "the function has more than 64 direct annotations",
        )

    output: list[RuntimeAnnotationLabel] = []
    for name, value in annotations.items():
        if type(name) is not str:
            raise TypeError("annotation names must be exact strings")
        if not 1 <= len(name) <= _MAX_ANNOTATION_NAME_CHARACTERS:
            raise RuntimeAnnotationProfileError(
                "an annotation name is outside the character limit",
            )
        if _SIMPLE_NAME.fullmatch(name) is None:
            raise RuntimeAnnotationProfileError(
                "annotation names must be simple ASCII identifiers",
            )

        if type(value) is str:
            raise RuntimeAnnotationProfileError(
                f"annotation {name} is a raw string rather than a ForwardRef"
            )
        if type(value) is ForwardRef:
            unresolved_name = _validate_dotted_label(
                value.__forward_arg__,
                field="forward reference",
            )
            output.append(RuntimeAnnotationLabel(name, unresolved_name, True))
            continue

        known_label = next(
            (entry.label for entry in checked_allowed if value is entry.value),
            None,
        )
        if known_label is not None:
            output.append(RuntimeAnnotationLabel(name, known_label, False))
            continue

        raise RuntimeAnnotationProfileError(
            f"annotation {name} is not an allowed identity or simple ForwardRef"
        )

    output.sort(key=lambda item: item.name)
    return tuple(output)


```

## Example

```python
annotation_events: list[str] = []


class LocalRecord:
    pass


def record_annotation_execution() -> type[LocalRecord]:
    annotation_events.append("annotation thunk ran")
    return LocalRecord


def transform(
    value: record_annotation_execution(),
    option: remote.Option,  # noqa: F821
) -> remote.Result:  # noqa: F821
    return value


allowed = (AllowedAnnotation(LocalRecord, "local.LocalRecord"),)
observed = inspect_trusted_runtime_annotations(transform, allowed)


def quoted(value: "remote.Option") -> None:  # noqa: F821, UP037
    pass


def nested(value: list[remote.Option]) -> None:  # noqa: F821
    pass


class NeverRepresent:
    def __repr__(self) -> str:
        raise AssertionError("arbitrary values must not be represented")


opaque_annotation = NeverRepresent()


def unsupported(value: opaque_annotation) -> None:
    pass


def rejected(function: FunctionType) -> bool:
    try:
        inspect_trusted_runtime_annotations(function, allowed)
    except RuntimeAnnotationProfileError:
        return True
    return False


assert (
    observed
    == (
        RuntimeAnnotationLabel("option", "remote.Option", True),
        RuntimeAnnotationLabel("return", "remote.Result", True),
        RuntimeAnnotationLabel("value", "local.LocalRecord", False),
    )
    and annotation_events == ["annotation thunk ran", "annotation thunk ran"]
    and rejected(quoted)
    and rejected(nested)
    and rejected(unsupported)
)
```

## Trade-offs and Limitations

After `get_annotations` returns, validation handles at most 64 direct entries
and 64 allowlist identities. Identity matching costs `O(A × N)` time for
allowlist size `A` and annotation count `N`, while sorting costs
`O(N log N)`; all retained names and labels have explicit character bounds.
No annotation value is hashed, compared for equality, evaluated as a
`ForwardRef`, or passed to `repr`.

Those post-call limits do not constrain annotation-thunk execution, including
repeated execution while `annotationlib` falls back to unresolved-name mode,
or the temporary objects, imports, side effects, runtime, and memory used before
the dictionary is returned. `eval_str=False` prevents explicitly quoted
strings from being evaluated; such strings are rejected rather than treated as
unresolved names. Stored forward-reference text is a runtime representation,
not a guarantee of exact source spelling; use static analysis when source
fidelity is required.
Nested references such as `list[Missing]`, unions, generic aliases, subclasses
of `ForwardRef`, non-ASCII names, and resolved objects absent from the identity
allowlist are also rejected. This is trusted runtime introspection, not static
type checking, sandboxing, authorization, or an untrusted-code safety boundary.

## Related Snippets

<!-- catalog:related:start -->
- [Extract a Bounded Static Python Annotation Index without Evaluation](../testing-tooling/extract-a-bounded-static-python-annotation-index-without-evaluation.md)
- [Check a Bounded Abstract Call Shape Against a Signature Without Invocation](check-a-bounded-abstract-call-shape-against-a-signature-without-invocation.md)
<!-- catalog:related:end -->
