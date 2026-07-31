---
title: "Initialize a Closed Cooperative super() Diamond by Consuming Keyword Arguments"
snippet_type: pattern
use_cases:
  - interoperability
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - compute-c3-linearizations-for-a-bounded-indexed-class-hierarchy.md
  - check-a-bounded-abstract-call-shape-against-a-signature-without-invocation.md
  - pass-constructor-only-context-with-initvar.md
---

# Initialize a Closed Cooperative super() Diamond by Consuming Keyword Arguments

## Idea and Problem

Initialize one closed multiple-inheritance diamond by letting each class consume only the keyword argument it owns.

`WorkItem(Named, Prioritized)` has one common `CooperativeRoot`. Each
initializer validates its keyword-only input, forwards the remaining keyword
arguments with zero-argument `super()`, and assigns its field only after the
rest of the MRO has initialized successfully. Here `super()` means the next
class in the instance's MRO; it does not mean one statically named parent.

The root closes the protocol by rejecting every unconsumed keyword before it
calls `object.__init__` through `super()` with no arguments. This prevents a
misspelled or unsupported option from disappearing silently.

## When to Use

Use this pattern for a small hierarchy that the same codebase owns completely
and whose mixins contribute independent validated state. It is useful when
either cooperative base order should initialize the shared root exactly once.

Every class that can appear in the MRO must follow the same convention: accept
cooperative keyword arguments, consume only its own keyword, and delegate once.
Prefer composition when a base is third-party, calls a named parent, does not
accept forwarded keywords, or has incompatible instance-layout requirements.

## Implementation

```python
_MAX_NAME_CHARACTERS = 64
_MAX_PRIORITY = 100


class CooperativeRoot:
    def __init__(self, **kwargs: object) -> None:
        if kwargs:
            unexpected = ", ".join(sorted(kwargs))
            raise TypeError(f"unexpected constructor arguments: {unexpected}")
        super().__init__()


class Named(CooperativeRoot):
    def __init__(self, *, name: str, **kwargs: object) -> None:
        if type(name) is not str:
            raise TypeError("name must be an exact string")
        if not 1 <= len(name) <= _MAX_NAME_CHARACTERS:
            raise ValueError("name length must be from 1 through 64 characters")

        super().__init__(**kwargs)
        self.name = name


class Prioritized(CooperativeRoot):
    def __init__(self, *, priority: int, **kwargs: object) -> None:
        if type(priority) is not int:
            raise TypeError("priority must be an exact non-boolean integer")
        if not 0 <= priority <= _MAX_PRIORITY:
            raise ValueError("priority must be from 0 through 100")

        super().__init__(**kwargs)
        self.priority = priority


class WorkItem(Named, Prioritized):
    pass
```

## Example

```python
class PriorityFirstWorkItem(Prioritized, Named):
    pass


assert WorkItem.__mro__ == (
    WorkItem,
    Named,
    Prioritized,
    CooperativeRoot,
    object,
)
assert PriorityFirstWorkItem.__mro__ == (
    PriorityFirstWorkItem,
    Prioritized,
    Named,
    CooperativeRoot,
    object,
)

named_first = WorkItem(name="review", priority=20)
priority_first = PriorityFirstWorkItem(name="publish", priority=80)

assert (named_first.name, named_first.priority) == ("review", 20)
assert (priority_first.name, priority_first.priority) == ("publish", 80)

type_failures = 0
for item_type, arguments in (
    (WorkItem, {"name": "missing-priority"}),
    (WorkItem, {"priority": 10}),
    (WorkItem, {"name": "unknown", "priority": 10, "owner": "team"}),
    (WorkItem, {"name": "boolean", "priority": True}),
):
    try:
        item_type(**arguments)
    except TypeError:
        type_failures += 1

value_failures = 0
for arguments in (
    {"name": "", "priority": 0},
    {"name": "x" * 65, "priority": 100},
    {"name": "low", "priority": -1},
    {"name": "high", "priority": 101},
):
    try:
        WorkItem(**arguments)
    except ValueError:
        value_failures += 1

unfinished = WorkItem.__new__(WorkItem)
try:
    WorkItem.__init__(unfinished, name="not-assigned", priority=True)
except TypeError:
    downstream_failure_rejected = True
else:
    downstream_failure_rejected = False

assert type_failures == 4
assert value_failures == 4
assert downstream_failure_rejected
assert not hasattr(unfinished, "name")
assert not hasattr(unfinished, "priority")
```

## Trade-offs and Limitations

Construction visits each participating initializer once in C3 MRO order, so
the delegation cost is linear in the bounded hierarchy depth. Reversing the
two cooperative bases changes which validator runs first and which field is
assigned first, but both orders reach the common root once and produce the same
two fields for valid input.

Validating before delegation rejects a bad value early. Assigning only after
delegation means an error deeper in the chain cannot leave an earlier layer's
field behind, as the partially initialized example demonstrates. This is not a
general transaction: setters, descriptors, or side effects added by another
class can still fail after a deeper layer has returned.

The name is an exact `str` of 1 through 64 Python characters and is not trimmed,
normalized, or encoded. Priority is an exact `int` from 0 through 100, so
booleans are rejected. Positional relays, non-cooperative third-party bases,
metaclasses, runtime class synthesis, extension-type layout conflicts, slots,
and dataclass-generated initialization are outside this closed teaching
pattern.

## Related Snippets

<!-- catalog:related:start -->
- [Compute C3 Linearizations for a Bounded Indexed Class Hierarchy](compute-c3-linearizations-for-a-bounded-indexed-class-hierarchy.md)
- [Check a Bounded Abstract Call Shape Against a Signature Without Invocation](check-a-bounded-abstract-call-shape-against-a-signature-without-invocation.md)
- [Pass Constructor-Only Context with dataclasses.InitVar](pass-constructor-only-context-with-initvar.md)
<!-- catalog:related:end -->
