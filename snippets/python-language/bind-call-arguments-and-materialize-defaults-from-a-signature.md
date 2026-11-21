---
title: "Bind Call Arguments and Materialize Defaults from a Signature"
snippet_type: recipe
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - check-a-bounded-abstract-call-shape-against-a-signature-without-invocation.md
  - ../testing-tooling/extract-function-default-expressions-from-python-source.md
---

# Bind Call Arguments and Materialize Defaults from a Signature

## Idea and Problem

Turn supplied call arguments into named parameter values that include any omitted defaults.

Tests and diagnostics often need to see which input a function would receive
regardless of whether a caller used a positional argument, a keyword, or an
omitted default. `Signature.bind()` applies Python's argument binding rules.
`apply_defaults()` fills in default values and adds empty variadic arguments
when none were supplied.

## When to Use

Use this when you have one trusted `Signature` and need a readable parameter
record for a proposed call. It fits test diagnostics or comparisons of known
inputs. The call must satisfy the whole signature, including positional-only
and keyword-only rules.

## Implementation

```python
from inspect import Signature


def materialize_arguments(
    call_signature: Signature, /, *args: object, **kwargs: object
) -> tuple[tuple[str, object], ...]:
    bound = call_signature.bind(*args, **kwargs)
    bound.apply_defaults()
    return tuple(bound.arguments.items())
```

## Example

```python
from inspect import signature


def render_report(
    label: str,
    /,
    width: int = 80,
    *columns: str,
    show_footer: bool = True,
    **options: object,
) -> str:
    raise AssertionError("the call should only be bound")


report_signature = signature(render_report)
implicit = materialize_arguments(report_signature, "daily")
explicit = materialize_arguments(report_signature, "daily", width=80, show_footer=True)

tags = ["draft"]
custom = materialize_arguments(
    report_signature, "daily", 60, "duration", show_footer=False, tags=tags
)
try:
    materialize_arguments(report_signature, label="daily")
except TypeError:
    positional_only_rejected = True
else:
    positional_only_rejected = False

defaults = (
    ("label", "daily"),
    ("width", 80),
    ("columns", ()),
    ("show_footer", True),
    ("options", {}),
)
assert implicit == explicit == defaults
assert custom == (
    ("label", "daily"),
    ("width", 60),
    ("columns", ("duration",)),
    ("show_footer", False),
    ("options", {"tags": tags}),
)
assert positional_only_rejected and dict(custom)["options"]["tags"] is tags
```

## Trade-offs and Limitations

The helper binds against the supplied signature. It does not choose a
signature or an overload, invoke the function, evaluate default expressions,
or enforce annotations. It uses the default objects already stored in the
signature. Introspect a live callable only when you trust its behavior
during introspection.

The outer tuples preserve parameter order. They do not freeze mutable
argument values or guarantee a hashable key. When testing behavior that
depends on argument identity or the order of extra keywords, preserve those
requirements explicitly.

## Related Snippets

<!-- catalog:related:start -->
- [Check a Bounded Abstract Call Shape Against a Signature Without Invocation](check-a-bounded-abstract-call-shape-against-a-signature-without-invocation.md)
- [Extract Function Default Expressions from Python Source](../testing-tooling/extract-function-default-expressions-from-python-source.md)
<!-- catalog:related:end -->
