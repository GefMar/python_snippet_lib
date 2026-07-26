---
title: "Validate Reused Fields with a Data Descriptor"
snippet_type: pattern
use_cases:
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - pass-constructor-only-context-with-initvar.md
---

# Validate Reused Fields with a Data Descriptor

## Idea and Problem

Centralize a repeated attribute rule in a data descriptor that validates every assignment and stores each field independently.

A descriptor with `__set__` controls assignment through a class attribute.
`__set_name__` receives the bound field name during class creation, allowing
each descriptor instance to choose a private storage name without repeating a
property for every managed attribute.

## When to Use

Use a data descriptor when the same compact validation and normalization rule
is reused across several attributes or classes and must run on reassignment as
well as construction. A single `property` is usually clearer for one field.
Instantiate a separate descriptor for every managed class attribute, and keep
rules with cross-field or external dependencies in the owning class instead.

## Implementation

```python
class NonBlankText:
    def __set_name__(self, owner: type[object], name: str) -> None:
        if hasattr(self, "_storage_name"):
            raise TypeError("a descriptor instance cannot manage multiple fields")
        self._public_name = name
        self._storage_name = f"_{name}"

    def __get__(
        self,
        instance: object | None,
        owner: type[object] | None = None,
    ) -> object:
        if instance is None:
            return self
        try:
            return getattr(instance, self._storage_name)
        except AttributeError:
            raise AttributeError(self._public_name) from None

    def __set__(self, instance: object, value: object) -> None:
        if not isinstance(value, str):
            raise TypeError(f"{self._public_name} must be text")
        normalized = value.strip()
        if not normalized:
            raise ValueError(f"{self._public_name} must not be blank")
        setattr(instance, self._storage_name, normalized)


class Article:
    title = NonBlankText()
    section = NonBlankText()

    def __init__(self, title: str, section: str) -> None:
        self.title = title
        self.section = section
```

## Example

```python
first = Article("  Overview  ", " Guides ")
second = Article("Reference", "API")
first.title = "Introduction"

try:
    first.section = "   "
except ValueError:
    blank_rejected = True
else:
    blank_rejected = False

try:
    Article(123, "Guides")
except TypeError:
    non_text_rejected = True
else:
    non_text_rejected = False

uninitialized = Article.__new__(Article)
try:
    uninitialized.title
except AttributeError as error:
    unset_name = str(error)
else:
    unset_name = ""

assert (
    first.title,
    first.section,
    second.title,
    Article.title is vars(Article)["title"],
    blank_rejected,
    non_text_rejected,
    unset_name,
) == ("Introduction", "Guides", "Reference", True, True, True, "title")
```

## Trade-offs and Limitations

Descriptors are less local than properties and can make attribute behavior
harder to discover. This storage strategy requires instances that permit the
generated private attributes; a slotted owner must declare compatible slots.
Assigning a descriptor after class creation does not invoke `__set_name__`
automatically. The implementation strips surrounding whitespace as policy and
rejects non-strings rather than coercing them; different rules should use a
different descriptor type.

## Related Snippets

<!-- catalog:related:start -->
- [Pass Constructor-Only Context with dataclasses.InitVar](pass-constructor-only-context-with-initvar.md)
<!-- catalog:related:end -->
