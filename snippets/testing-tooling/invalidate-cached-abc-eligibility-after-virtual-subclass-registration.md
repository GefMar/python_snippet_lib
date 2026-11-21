---
title: "Invalidate Cached ABC Eligibility After Virtual Subclass Registration"
snippet_type: testing-technique
use_cases:
  - caching
  - interoperability
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/dispatch-on-first-argument-type-with-functools-singledispatch.md
  - ../python-language/recompute-abstract-methods-after-installing-implementations-with-a-class-decorator.md
---

# Invalidate Cached ABC Eligibility After Virtual Subclass Registration

## Idea and Problem

Discard stale application cache entries when a virtual subclass is registered.

`ABC.register()` can make an earlier negative `issubclass()` result become
positive. Python updates its own ABC checks, but it cannot clear your
application's cache. Compare the current `get_cache_token()` with the previous
one and clear your cache when they differ.

## When to Use

Use this for an existing cache of decisions that depend on ABC membership,
such as selecting a handler family after optional adapters are registered.
The bare subclass check below models that dependency; if you only need
`issubclass()`, call it directly.

## Implementation

```python
from abc import ABC, abstractmethod, get_cache_token
from functools import lru_cache


class Reporter(ABC):
    @abstractmethod
    def report(self, message: str) -> str: ...


_registry_token = get_cache_token()


@lru_cache(maxsize=128)
def _cached_eligibility(candidate: type) -> bool:
    return issubclass(candidate, Reporter)


def is_reporter(candidate: type) -> bool:
    global _registry_token
    current_token = get_cache_token()
    if current_token != _registry_token:
        _cached_eligibility.cache_clear()
        _registry_token = current_token
    return _cached_eligibility(candidate)
```

## Example

```python
class ExternalReporter:
    def report(self, message: str) -> str:
        return message.upper()


assert is_reporter(ExternalReporter) is False
Reporter.register(ExternalReporter)

assert issubclass(ExternalReporter, Reporter) is True
assert _cached_eligibility(ExternalReporter) is False
assert is_reporter(ExternalReporter) is True
assert _cached_eligibility(ExternalReporter) is True
```

## Trade-offs and Limitations

The token is shared across ABCs. Registration on an unrelated ABC can also
clear this small cache. Compare tokens for equality; do not assume they are
ordered counters. If a decision also depends on mutable configuration, that
state needs its own invalidation rule.

The check and cache clear are not atomic with registration. This example is
for sequential use, such as registrations during startup or a test. Virtual
registration declares membership; it does not supply methods or verify
behavior. For ordinary type-based function dispatch, prefer
`functools.singledispatch()` rather than building another dispatch cache.

## Related Snippets

<!-- catalog:related:start -->
- [Dispatch on First-Argument Type with functools.singledispatch](../python-language/dispatch-on-first-argument-type-with-functools-singledispatch.md)
- [Recompute Abstract Methods After Installing Implementations with a Class Decorator](../python-language/recompute-abstract-methods-after-installing-implementations-with-a-class-decorator.md)
<!-- catalog:related:end -->
