---
title: "Bind Non-Leading Positional Arguments with functools.Placeholder"
snippet_type: idiom
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - read-fixed-size-blocks-with-iter-sentinel.md
  - check-a-bounded-abstract-call-shape-against-a-signature-without-invocation.md
---

# Bind Non-Leading Positional Arguments with functools.Placeholder

## Idea and Problem

Bind a non-leading positional argument directly while leaving earlier positions to be supplied when the resulting callable is invoked.

Python 3.14 adds `functools.Placeholder` as a positional hole inside
`partial`. Call-time arguments fill stored holes from left to right. Once every
hole is filled, any remaining positional arguments are appended after the
stored argument template.

Here, one partial fixes the middle exponent while leaving the base as a hole;
the later modulus is therefore appended. A second partial fixes the trailing
modulus and leaves two ordered holes for the base and exponent.

## When to Use

Use this idiom when an API expects a narrower callback but the value to freeze
is not the first positional argument of an existing callable. It avoids a
single-purpose lambda while retaining an inspectable `partial` object and the
wrapped callable's validation.

Prefer an ordinary leading-argument partial when no holes are needed, and use a
named function when adaptation needs branching, transformation, documentation,
or side-effect policy. `Placeholder` requires Python 3.14 and does not add
runtime type checking of its own.

## Implementation

```python
from functools import Placeholder, partial

_MAX_MODULAR_EXPONENT = 1_000_000
_MAX_MODULUS = (1 << 63) - 1


def bounded_modular_power(base: int, exponent: int, modulus: int) -> int:
    """Return a validated bounded modular power."""
    if type(base) is not int:
        raise TypeError("base must be an exact integer")
    if type(exponent) is not int:
        raise TypeError("exponent must be an exact integer")
    if type(modulus) is not int:
        raise TypeError("modulus must be an exact integer")
    if not 2 <= modulus <= _MAX_MODULUS:
        raise ValueError("modulus is outside 2..2**63-1")
    if not 0 <= base < modulus:
        raise ValueError("base must be canonical modulo modulus")
    if not 0 <= exponent <= _MAX_MODULAR_EXPONENT:
        raise ValueError("exponent is outside 0..1000000")
    return pow(base, exponent, modulus)


fifth_power_modulo = partial(
    bounded_modular_power,
    Placeholder,
    5,
)
power_modulo_257 = partial(
    bounded_modular_power,
    Placeholder,
    Placeholder,
    257,
)
```

## Example

```python
def raises(error_type: type[Exception], operation) -> bool:
    try:
        operation()
    except error_type:
        return True
    return False


checked = 0
for base in (0, 1, 2, 19, 256):
    for exponent in (0, 1, 2, 17, _MAX_MODULAR_EXPONENT):
        assert power_modulo_257(base, exponent) == bounded_modular_power(
            base,
            exponent,
            257,
        )
        checked += 1

appended_moduli_checked = 0
for modulus in (2, 3, 97, 65_537, _MAX_MODULUS):
    base = min(7, modulus - 1)
    assert fifth_power_modulo(base, modulus) == bounded_modular_power(
        base,
        5,
        modulus,
    )
    appended_moduli_checked += 1

assert (
    checked == 25
    and appended_moduli_checked == 5
    and power_modulo_257(3, 11) == pow(3, 11, 257)
    and raises(TypeError, lambda: power_modulo_257(3))
    and raises(TypeError, lambda: power_modulo_257())
    and raises(TypeError, lambda: power_modulo_257(3, 11, 5))
    and raises(
        TypeError,
        lambda: partial(bounded_modular_power, base=Placeholder),
    )
    and raises(TypeError, lambda: fifth_power_modulo(True, 17))
    and raises(ValueError, lambda: fifth_power_modulo(17, 17))
)
```

## Trade-offs and Limitations

Constructing and calling each `partial` adds `O(1)` wrapper work and retains
references to the wrapped callable, fixed values, and positional template. The
wrapped modular exponentiation owns the runtime cost, which is logarithmic in
the bounded exponent with operand-dependent Python integer arithmetic.

Every placeholder must be filled at call time, in left-to-right order.
Additional positional arguments are appended after the stored template, while
`Placeholder` is forbidden as a keyword value. Nested `partial` composition and
its hole-rewriting behavior are outside this example. The idiom does not
validate annotations automatically, constrain side effects, or provide
compatibility with Python versions before 3.14.

## Related Snippets

<!-- catalog:related:start -->
- [Read Fixed-Size Blocks with iter() and a Sentinel](read-fixed-size-blocks-with-iter-sentinel.md)
- [Check a Bounded Abstract Call Shape Against a Signature Without Invocation](check-a-bounded-abstract-call-shape-against-a-signature-without-invocation.md)
<!-- catalog:related:end -->
