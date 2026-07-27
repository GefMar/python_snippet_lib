---
title: "Load a Bounded Allocation Rule with Marshmallow Cross-Field Validation"
snippet_type: integration
use_cases:
  - configuration
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: marshmallow
    version: "4.3.0"
related:
  - migrate-one-bounded-json-record-to-a-current-version.md
  - reject-unknown-options-with-conservative-typo-suggestions.md
  - ../python-language/validate-reused-fields-with-a-data-descriptor.md
---

# Load a Bounded Allocation Rule with Marshmallow Cross-Field Validation

## Idea and Problem

Load one closed allocation-rule mapping through strict Marshmallow fields, deterministic cross-field validation, and a frozen result type.

Custom fields reject coercion and Python's `bool`-as-`int` behavior before a
schema-level hook checks ordering and step alignment. Every field is required,
unknown fields raise errors, and the hook deliberately skips records with field
errors so incomplete data cannot cause incidental `KeyError` exceptions.

## When to Use

Use this integration at an in-memory configuration boundary where callers
already hold one dictionary and need Marshmallow's field-addressed error
structure. Pin Marshmallow because decorator call conventions and default error
messages are part of the tested behavior. Route every construction through
`load_allocation_rule` when the returned dataclass must satisfy the schema.

Keep file decoding, environment expansion, and application-specific allocation
decisions outside this loader. Use a direct dataclass validator for a tiny
dependency-free boundary, or separate domain validation when rules evolve
independently of the serialized mapping.

## Implementation

```python
import re
from dataclasses import dataclass
from typing import cast

from marshmallow import RAISE, Schema, ValidationError, fields, post_load, validates_schema


_NAME = re.compile(r"[a-z][a-z0-9_-]{0,31}\Z", re.ASCII)
_FIELD_NAME = re.compile(r"[a-z][a-z0-9_]{0,31}\Z", re.ASCII)
_MAX_ALLOCATION = 1_000_000
_MAX_INPUT_FIELDS = 5


@dataclass(frozen=True, slots=True)
class AllocationRule:
    name: str
    minimum: int
    preferred: int
    maximum: int
    step: int


class StrictName(fields.Field):
    def _deserialize(
        self,
        value: object,
        attr: str | None,
        data: object,
        **kwargs: object,
    ) -> str:
        if type(value) is not str:
            raise ValidationError("Must be an exact string.")
        if _NAME.fullmatch(value) is None:
            raise ValidationError("Must be a canonical ASCII token of 1 to 32 characters.")
        return value


class StrictBoundedInteger(fields.Field):
    def __init__(
        self,
        *,
        minimum: int,
        maximum: int,
        **kwargs: object,
    ) -> None:
        super().__init__(**kwargs)
        self.minimum = minimum
        self.maximum = maximum

    def _deserialize(
        self,
        value: object,
        attr: str | None,
        data: object,
        **kwargs: object,
    ) -> int:
        if type(value) is not int:
            raise ValidationError("Must be an exact integer.")
        if not self.minimum <= value <= self.maximum:
            raise ValidationError("Must be within the allowed range.")
        return value


class AllocationRuleSchema(Schema):
    name = StrictName(required=True)
    minimum = StrictBoundedInteger(
        required=True,
        minimum=0,
        maximum=_MAX_ALLOCATION,
    )
    preferred = StrictBoundedInteger(
        required=True,
        minimum=0,
        maximum=_MAX_ALLOCATION,
    )
    maximum = StrictBoundedInteger(
        required=True,
        minimum=0,
        maximum=_MAX_ALLOCATION,
    )
    step = StrictBoundedInteger(
        required=True,
        minimum=1,
        maximum=_MAX_ALLOCATION,
    )

    @validates_schema(skip_on_field_errors=True)
    def validate_relationships(
        self,
        data: dict[str, object],
        **kwargs: object,
    ) -> None:
        minimum = cast(int, data["minimum"])
        preferred = cast(int, data["preferred"])
        maximum = cast(int, data["maximum"])
        step = cast(int, data["step"])

        ordering_errors: dict[str, list[str]] = {}
        if minimum > preferred:
            ordering_errors["preferred"] = [
                "Must be greater than or equal to minimum."
            ]
        if preferred > maximum:
            ordering_errors["maximum"] = [
                "Must be greater than or equal to preferred."
            ]
        if ordering_errors:
            raise ValidationError(ordering_errors)

        alignment_errors: dict[str, list[str]] = {}
        if (preferred - minimum) % step != 0:
            alignment_errors["preferred"] = [
                "Must be aligned to minimum by step."
            ]
        if (maximum - minimum) % step != 0:
            alignment_errors["maximum"] = [
                "Must be aligned to minimum by step."
            ]
        if alignment_errors:
            raise ValidationError(alignment_errors)

    @post_load
    def make_rule(
        self,
        data: dict[str, object],
        **kwargs: object,
    ) -> AllocationRule:
        return AllocationRule(
            name=cast(str, data["name"]),
            minimum=cast(int, data["minimum"]),
            preferred=cast(int, data["preferred"]),
            maximum=cast(int, data["maximum"]),
            step=cast(int, data["step"]),
        )


def load_allocation_rule(document: dict[str, object]) -> AllocationRule:
    if type(document) is not dict:
        raise ValidationError({"_schema": ["Input must be an exact mapping."]})
    if len(document) > _MAX_INPUT_FIELDS:
        raise ValidationError({"_schema": ["Input contains too many fields."]})
    if any(
        type(key) is not str or _FIELD_NAME.fullmatch(key) is None
        for key in document
    ):
        raise ValidationError({"_schema": ["Field names must be bounded ASCII tokens."]})
    snapshot = dict(document)
    result = AllocationRuleSchema(unknown=RAISE).load(snapshot)
    return cast(AllocationRule, result)
```

## Example

```python
from dataclasses import FrozenInstanceError


source = {
    "name": "worker_pool",
    "minimum": 10,
    "preferred": 30,
    "maximum": 50,
    "step": 20,
}
before = dict(source)
rule = load_allocation_rule(source)

try:
    rule.step = 1
except FrozenInstanceError:
    result_is_frozen = True
else:
    result_is_frozen = False

assert rule == AllocationRule("worker_pool", 10, 30, 50, 20)
assert source == before
assert result_is_frozen

submitted_preferred = 35
try:
    load_allocation_rule({**source, "preferred": submitted_preferred})
except ValidationError as error:
    errors = error.messages
else:
    raise AssertionError("misaligned rule unexpectedly loaded")

assert errors == {"preferred": ["Must be aligned to minimum by step."]}
assert str(submitted_preferred) not in repr(errors)
```

## Trade-offs and Limitations

The wrapper bounds the top-level dictionary at five exact string keys, copies
it, and returns only immutable scalar fields, so it neither mutates nor retains
the caller's mapping. It does not decode files, YAML, environment variables,
or domain-specific allocation policies. Direct `AllocationRule` construction
bypasses the schema and should not be exposed as an alternative input boundary
when these invariants matter.

Field failures take precedence over cross-field failures because the schema
hook uses `skip_on_field_errors=True`. Ordering failures likewise take
precedence over alignment failures, keeping diagnostics deterministic but
reporting fewer issues in one pass. Error messages deliberately omit submitted
values; callers should preserve that property when presenting or logging the
field-addressed error mapping.

## Related Snippets

<!-- catalog:related:start -->
- [Migrate One Bounded JSON Record to a Current Version](migrate-one-bounded-json-record-to-a-current-version.md)
- [Reject Unknown Options with Conservative Typo Suggestions](reject-unknown-options-with-conservative-typo-suggestions.md)
- [Validate Reused Fields with a Data Descriptor](../python-language/validate-reused-fields-with-a-data-descriptor.md)
<!-- catalog:related:end -->
