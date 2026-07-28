---
title: "Split a Bounded ExceptionGroup by Closed Exception Types"
snippet_type: pattern
use_cases:
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - add-bounded-stage-context-without-replacing-an-exception.md
  - ../concurrency-lifecycle/own-one-external-mount-lease-with-exception-safe-cleanup.md
  - ../observability-operations/capture-a-bounded-pickle-friendly-exception-report.md
---

# Split a Bounded ExceptionGroup by Closed Exception Types

## Idea and Problem

Preflight a built-in exception-group tree against explicit structural limits, then delegate partitioning to the standard-library split operation with a closed tuple of exception classes.

The standard-library operation preserves the original tree shape and exception
metadata. The wrapper bounds the work first and excludes callable predicates,
custom group subclasses, and unvalidated match conditions without rebuilding
the split algorithm.

## When to Use

Use this pattern at a trusted in-process boundary when an exception group must
be divided into handled and unhandled parts and both the accepted classes and
maximum tree size are policy decisions. The root and every nested group must be
an exact built-in `BaseExceptionGroup` or `ExceptionGroup`.

Matching deliberately follows normal `except` semantics, not exact-type
equality: requesting a base exception class also matches its subclasses. The
standard library checks group nodes as well as leaves, so a class that matches
a group node can select that whole subtree. Use separate explicit tuples when
different handlers need different closed policies.

## Implementation

```python
_MAX_MATCH_TYPES = 16
_MAX_GROUP_DEPTH = 32
_MAX_GROUP_NODES = 256
_MAX_LEAF_EXCEPTIONS = 1024
_BUILTIN_GROUP_TYPES = (BaseExceptionGroup, ExceptionGroup)


class ExceptionGroupLimitError(ValueError):
    pass


def _validated_exception_types(
    value: object,
) -> tuple[type[BaseException], ...]:
    if type(value) is not tuple:
        raise TypeError("match_types must be an exact tuple")
    if not 1 <= len(value) <= _MAX_MATCH_TYPES:
        raise ValueError("match_types count is outside the supported range")

    checked: list[type[BaseException]] = []
    for item in value:
        if not isinstance(item, type) or not issubclass(item, BaseException):
            raise TypeError("every match type must be an exception class")
        if any(item is previous for previous in checked):
            raise ValueError("match_types must not contain duplicate classes")
        checked.append(item)
    return tuple(checked)


def _preflight_exception_group(
    value: object,
) -> BaseExceptionGroup:
    if type(value) not in _BUILTIN_GROUP_TYPES:
        raise TypeError(
            "group must be an exact BaseExceptionGroup or ExceptionGroup"
        )

    pending: list[tuple[BaseExceptionGroup, int]] = [(value, 1)]
    group_nodes = 0
    leaf_exceptions = 0

    while pending:
        group, depth = pending.pop()
        if type(group) not in _BUILTIN_GROUP_TYPES:
            raise TypeError("every nested group must use a built-in group type")
        if depth > _MAX_GROUP_DEPTH:
            raise ExceptionGroupLimitError(
                "exception group nesting exceeds the supported depth"
            )

        group_nodes += 1
        if group_nodes > _MAX_GROUP_NODES:
            raise ExceptionGroupLimitError(
                "exception group node count exceeds the supported limit"
            )

        for child in group.exceptions:
            if isinstance(child, BaseExceptionGroup):
                pending.append((child, depth + 1))
            else:
                leaf_exceptions += 1
                if leaf_exceptions > _MAX_LEAF_EXCEPTIONS:
                    raise ExceptionGroupLimitError(
                        "leaf exception count exceeds the supported limit"
                    )
    return value


def split_bounded_exception_group(
    group: BaseExceptionGroup,
    match_types: tuple[type[BaseException], ...],
) -> tuple[BaseExceptionGroup | None, BaseExceptionGroup | None]:
    types = _validated_exception_types(match_types)
    checked_group = _preflight_exception_group(group)
    return checked_group.split(types)
```

## Example

```python
def leaves(group: BaseExceptionGroup | None) -> tuple[BaseException, ...]:
    if group is None:
        return ()
    pending: list[BaseException] = [group]
    result: list[BaseException] = []
    while pending:
        item = pending.pop()
        if isinstance(item, BaseExceptionGroup):
            pending.extend(reversed(item.exceptions))
        else:
            result.append(item)
    return tuple(result)


value_error = ValueError("private value")
try:
    raise KeyError("private key")
except KeyError as caught:
    key_error = caught
timeout_error = TimeoutError("private timeout")

group = ExceptionGroup(
    "operation failed",
    (
        value_error,
        ExceptionGroup("nested failures", (key_error, timeout_error)),
    ),
)
group.add_note("bounded operation")
key_traceback = key_error.__traceback__

no_match, unchanged = split_bounded_exception_group(group, (SyntaxError,))
full_match, no_remainder = split_bounded_exception_group(group, (Exception,))
lookup_match, other = split_bounded_exception_group(group, (LookupError,))

assert (
    no_match,
    leaves(unchanged),
    leaves(full_match),
    no_remainder,
    leaves(lookup_match),
    leaves(other),
    lookup_match.__notes__ if lookup_match is not None else None,
    leaves(lookup_match)[0].__traceback__ is key_traceback,
) == (
    None,
    (value_error, key_error, timeout_error),
    (value_error, key_error, timeout_error),
    None,
    (key_error,),
    (value_error, timeout_error),
    ["bounded operation"],
    True,
)
```

## Trade-offs and Limitations

Preflight is iterative and linear in at most 256 group nodes and 1,024 leaf
exceptions, with root depth defined as one. The limits bound traversal before
the standard-library split begins; the split itself still allocates derived
group nodes. Exact built-in group types keep the wrapper's ordinary validation
failures closed and avoid invoking a custom group's `derive()` implementation.

`BaseExceptionGroup.split()` preserves matched tree structure, group messages,
notes, causes, contexts, and tracebacks according to standard-library rules.
It also retains the original leaf exception objects and their traceback
objects. Those objects can retain frames, locals, exception messages, and
other secrets, so this is not a sanitization or serialization boundary. The
wrapper does not log, format, redact, mutate, raise, or handle either result.

## Related Snippets

<!-- catalog:related:start -->
- [Add Bounded Stage Context Without Replacing an Exception](add-bounded-stage-context-without-replacing-an-exception.md)
- [Own One External Mount Lease with Exception-Safe Cleanup](../concurrency-lifecycle/own-one-external-mount-lease-with-exception-safe-cleanup.md)
- [Capture a Bounded Pickle-Friendly Exception Report](../observability-operations/capture-a-bounded-pickle-friendly-exception-report.md)
<!-- catalog:related:end -->
