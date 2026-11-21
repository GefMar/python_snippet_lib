---
title: "Recompute Abstract Methods After Installing Implementations with a Class Decorator"
snippet_type: recipe
use_cases:
  - interoperability
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - collect-decorated-methods-in-class-definition-order.md
  - initialize-a-closed-cooperative-super-diamond-by-consuming-keyword-arguments.md
---

# Recompute Abstract Methods After Installing Implementations with a Class Decorator

## Idea and Problem

Make an abstract subclass instantiable when a class decorator supplies the missing implementation after class creation.

`ABCMeta` determines the abstract methods when it creates the class. A
class decorator runs afterward. Assign the concrete method and call
`update_abstractmethods()` on the class so Python reconsiders its abstract
status before the decorator returns.

## When to Use

Use this when a class decorator intentionally installs a method on an ABC
subclass you own, such as a formatter assembled from a fixed rendering
function. Prefer an ordinary method in the class body when no deferred
installation is needed.

## Implementation

```python
from abc import ABC, abstractmethod, update_abstractmethods


class Renderer(ABC):
    @abstractmethod
    def render(self, text: str) -> str:
        raise NotImplementedError


def bracket_render(self: Renderer, text: str) -> str:
    return f"[{text}]"


def install_bracket_renderer[T: Renderer](cls: type[T]) -> type[T]:
    cls.render = bracket_render
    return update_abstractmethods(cls)
```

## Example

```python
from inspect import isabstract


@install_bracket_renderer
class BracketRenderer(Renderer):
    pass


class LateRenderer(Renderer):
    pass


class ExistingChild(LateRenderer):
    pass


LateRenderer.render = bracket_render
assignment_only = isabstract(LateRenderer)
update_abstractmethods(LateRenderer)
child_still_abstract = isabstract(ExistingChild)
update_abstractmethods(ExistingChild)

assert (
    BracketRenderer().render("ready"),
    assignment_only,
    LateRenderer().render("late"),
    child_still_abstract,
    ExistingChild().render("child"),
) == ("[ready]", True, "[late]", True, "[child]")
```

## Trade-offs and Limitations

The update affects only the supplied class. If existing descendants also
need an update, refresh the affected classes explicitly with their bases
first. Classes created later compute their own abstract methods normally.

Recomputing abstract methods verifies abstract status, not implementation
signatures or results. Keep the installed method compatible with the
declared interface and test its behavior. The example modifies only its
own classes.

## Related Snippets

<!-- catalog:related:start -->
- [Collect Decorated Methods in Class Definition Order](collect-decorated-methods-in-class-definition-order.md)
- [Initialize a Closed Cooperative super() Diamond by Consuming Keyword Arguments](initialize-a-closed-cooperative-super-diamond-by-consuming-keyword-arguments.md)
<!-- catalog:related:end -->
