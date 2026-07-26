---
title: "Dispatch Named Strategies with an Explicit Function Mapping"
snippet_type: pattern
use_cases:
  - configuration
  - interoperability
tested_python:
  - "3.14"
dependencies: []
related: []
---

# Dispatch Named Strategies with an Explicit Function Mapping

## Idea and Problem

Select stateless behavior by name with an explicit read-only mapping instead of a class hierarchy or hidden global registry.

The registry builder copies the supplied mapping and exposes a read-only view,
so later mutations of the source dictionary cannot change dispatch behavior.
The dispatcher looks up one callable, forwards arguments unchanged, and raises
a dedicated error when configuration names an unavailable strategy.

## When to Use

Use this pattern when several small functions implement the same conceptual
signature and a trusted configuration value chooses one of them. Construct the
mapping at an explicit composition point where all strategies are imported.
Prefer direct calls when there are only one or two fixed paths and no runtime
selection is needed.

## Implementation

```python
from collections.abc import Callable, Mapping
from types import MappingProxyType
from typing import ParamSpec, TypeVar


Arguments = ParamSpec("Arguments")
ResultT = TypeVar("ResultT")


class UnknownStrategyError(LookupError):
    pass


def freeze_strategies(
    strategies: Mapping[str, Callable[Arguments, ResultT]],
) -> Mapping[str, Callable[Arguments, ResultT]]:
    copied: dict[str, Callable[Arguments, ResultT]] = {}
    for name, strategy in strategies.items():
        if not isinstance(name, str) or not name:
            raise ValueError("strategy names must be non-empty strings")
        if not callable(strategy):
            raise TypeError(f"strategy {name!r} must be callable")
        copied[name] = strategy
    return MappingProxyType(copied)


def dispatch_strategy(
    name: str,
    strategies: Mapping[str, Callable[Arguments, ResultT]],
    /,
    *args: Arguments.args,
    **kwargs: Arguments.kwargs,
) -> ResultT:
    try:
        strategy = strategies[name]
    except KeyError as error:
        raise UnknownStrategyError(f"unknown strategy: {name!r}") from error
    return strategy(*args, **kwargs)
```

## Example

```python
def repeat(text: str, *, count: int) -> str:
    return text * count


def surround(text: str, *, count: int) -> str:
    return ("[" * count) + text + ("]" * count)


source = {"repeat": repeat, "surround": surround}
strategies = freeze_strategies(source)
source["late"] = repeat

repeated = dispatch_strategy("repeat", strategies, "ha", count=2)
surrounded = dispatch_strategy("surround", strategies, "value", count=1)

try:
    dispatch_strategy("missing", strategies, "x", count=1)
except UnknownStrategyError:
    missing_rejected = True
else:
    missing_rejected = False

try:
    strategies["other"] = repeat
except TypeError:
    mutation_rejected = True
else:
    mutation_rejected = False

assert (
    repeated,
    surrounded,
    "late" in strategies,
    missing_rejected,
    mutation_rejected,
) == ("haha", "[value]", False, True, True)
```

## Trade-offs and Limitations

All functions should obey one semantic signature, but runtime dispatch cannot
prove that independently authored callables accept the forwarded arguments.
The wrapper deliberately adds no retries, lifecycle, dependency injection, or
error translation; strategy exceptions propagate. It is not plugin discovery:
external plugins need an explicit loading, validation, compatibility, and
trust policy rather than import-time registration side effects.

## Related Snippets

<!-- catalog:related:start -->
No related snippets.
<!-- catalog:related:end -->
