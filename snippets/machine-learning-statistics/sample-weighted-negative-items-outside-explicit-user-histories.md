---
title: "Sample Weighted Negative Items Outside Explicit User Histories"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/sample-a-weighted-stream-with-a-fixed-size-reservoir.md
  - ../data-processing/sample-stream-items-independently-with-a-fixed-probability.md
  - fit-and-apply-an-exact-categorical-frequency-encoder.md
---

# Sample Weighted Negative Items Outside Explicit User Histories

## Idea and Problem

Draw reproducible weighted negative items with replacement from a frozen universe while excluding each user's explicit history.

Integer weights define an exact cumulative distribution over the eligible
items for each user. A local seeded random generator draws integer tickets
from that distribution, so no global random state or rejection loop is
involved. The signed 64-bit seed is validated before use. Frozen request and
result objects make the user order and sampled values explicit.

## When to Use

Use this algorithm when a bounded training or evaluation step already has a
reviewed item universe, fixed non-negative integer weights, and complete user
histories for that universe. It is suitable when repeated negative items are
allowed and exact replay on the tested Python version is more important than
estimating weights inside the sampler.

Fit popularity or other weights in a separate training-only step, freeze the
result, and pass it here. Use a specialized recommender library when sampling
must be without replacement, distributed across workers, conditioned on more
features, or statistically stable across Python implementations.

## Implementation

```python
from bisect import bisect_left
from dataclasses import dataclass
from random import Random


_MAX_IDENTIFIER_BYTES = 256
_MAX_UNIVERSE_ITEMS = 10_000
_MAX_USERS = 2_000
_MAX_HISTORY_ENTRIES = 200_000
_MAX_REQUESTED_SAMPLES = 100_000
_MAX_USER_ITEM_CHECKS = 1_000_000
_MAX_ITEM_WEIGHT = (1 << 31) - 1
_MAX_CUMULATIVE_WEIGHT = (1 << 63) - 1
_MIN_SEED = -(1 << 63)
_MAX_SEED = (1 << 63) - 1


def _negative_identifier(value: object, *, name: str) -> str:
    if type(value) is not str:
        raise TypeError(f"{name} must be text")
    if not value or value != value.strip() or not value.isprintable():
        raise ValueError(f"{name} must be non-empty trimmed printable text")
    if len(value.encode("utf-8")) > _MAX_IDENTIFIER_BYTES:
        raise ValueError(f"{name} exceeds the UTF-8 byte limit")
    return value


@dataclass(frozen=True, slots=True)
class WeightedItem:
    item_id: str
    weight: int

    def __post_init__(self) -> None:
        _negative_identifier(self.item_id, name="item_id")
        if type(self.weight) is not int:
            raise TypeError("weight must be an integer")
        if not 0 <= self.weight <= _MAX_ITEM_WEIGHT:
            raise ValueError("weight is outside the supported range")


@dataclass(frozen=True, slots=True)
class NegativeRequest:
    user_id: str
    history: tuple[str, ...]
    count: int

    def __post_init__(self) -> None:
        _negative_identifier(self.user_id, name="user_id")
        if type(self.history) is not tuple:
            raise TypeError("history must be a tuple")
        if len(self.history) > _MAX_HISTORY_ENTRIES:
            raise ValueError("history entries exceed the supported limit")
        for item_id in self.history:
            _negative_identifier(item_id, name="history item")
        if len(set(self.history)) != len(self.history):
            raise ValueError("history must not contain duplicate items")
        if type(self.count) is not int:
            raise TypeError("count must be an integer")
        if not 0 <= self.count <= _MAX_REQUESTED_SAMPLES:
            raise ValueError("count is outside the supported range")


@dataclass(frozen=True, slots=True)
class NegativeItems:
    user_id: str
    item_ids: tuple[str, ...]


def sample_weighted_negative_items(
    universe: tuple[WeightedItem, ...],
    requests: tuple[NegativeRequest, ...],
    *,
    seed: int,
) -> tuple[NegativeItems, ...]:
    if type(universe) is not tuple:
        raise TypeError("universe must be a tuple")
    if not 1 <= len(universe) <= _MAX_UNIVERSE_ITEMS:
        raise ValueError("universe size is outside the supported range")
    if type(requests) is not tuple:
        raise TypeError("requests must be a tuple")
    if len(requests) > _MAX_USERS:
        raise ValueError("user count exceeds the supported limit")
    if type(seed) is not int:
        raise TypeError("seed must be an integer")
    if not _MIN_SEED <= seed <= _MAX_SEED:
        raise ValueError("seed is outside the supported range")

    item_ids: set[str] = set()
    cumulative_universe_weight = 0
    for item in universe:
        if type(item) is not WeightedItem:
            raise TypeError("universe must contain WeightedItem values")
        if item.item_id in item_ids:
            raise ValueError("universe must not contain duplicate items")
        item_ids.add(item.item_id)
        cumulative_universe_weight += item.weight
        if cumulative_universe_weight > _MAX_CUMULATIVE_WEIGHT:
            raise ValueError("cumulative item weight exceeds the supported limit")

    if len(universe) * len(requests) > _MAX_USER_ITEM_CHECKS:
        raise ValueError("user-item eligibility work exceeds the supported limit")

    user_ids: set[str] = set()
    history_entries = 0
    requested_samples = 0
    for request in requests:
        if type(request) is not NegativeRequest:
            raise TypeError("requests must contain NegativeRequest values")
        history_entries += len(request.history)
        if history_entries > _MAX_HISTORY_ENTRIES:
            raise ValueError("history entries exceed the supported limit")
        requested_samples += request.count
        if requested_samples > _MAX_REQUESTED_SAMPLES:
            raise ValueError("requested samples exceed the supported limit")
        if request.user_id in user_ids:
            raise ValueError("requests must not contain duplicate users")
        user_ids.add(request.user_id)
        unknown = set(request.history).difference(item_ids)
        if unknown:
            raise ValueError("a user history contains an unknown item")

    rng = Random(seed)
    results: list[NegativeItems] = []
    for request in requests:
        seen = set(request.history)
        eligible_items: list[str] = []
        cumulative_weights: list[int] = []
        eligible_weight = 0
        for item in universe:
            if item.item_id in seen or item.weight == 0:
                continue
            eligible_weight += item.weight
            eligible_items.append(item.item_id)
            cumulative_weights.append(eligible_weight)

        if eligible_weight == 0:
            raise ValueError("a user has no eligible item with positive weight")

        if request.count == 0:
            results.append(NegativeItems(request.user_id, ()))
            continue

        sampled = tuple(
            eligible_items[
                bisect_left(
                    cumulative_weights,
                    rng.randrange(eligible_weight) + 1,
                )
            ]
            for _ in range(request.count)
        )
        results.append(NegativeItems(request.user_id, sampled))

    return tuple(results)
```

## Example

```python
universe = (
    WeightedItem("alpha", 1),
    WeightedItem("beta", 3),
    WeightedItem("gamma", 6),
    WeightedItem("inactive", 0),
)
requests = (
    NegativeRequest("user-a", ("alpha",), 5),
    NegativeRequest("user-b", ("beta", "gamma"), 3),
    NegativeRequest("user-c", ("alpha",), 0),
)

first = sample_weighted_negative_items(universe, requests, seed=23)
replayed = sample_weighted_negative_items(universe, requests, seed=23)

try:
    sample_weighted_negative_items(
        (WeightedItem("alpha", 1), WeightedItem("alpha", 2)),
        (),
        seed=1,
    )
except ValueError:
    duplicate_rejected = True
else:
    duplicate_rejected = False

assert first == replayed
assert first == (
    NegativeItems("user-a", ("gamma", "beta", "beta", "gamma", "gamma")),
    NegativeItems("user-b", ("alpha", "alpha", "alpha")),
    NegativeItems("user-c", ()),
)
assert duplicate_rejected
```

## Trade-offs and Limitations

Eligibility construction costs `O(users * items)`, and drawing costs
`O(samples * log eligible_items)`. The explicit caps bound both work and
memory. Histories and the universe must be complete and internally consistent;
an unknown history item is rejected instead of being silently ignored.

Sampling is with replacement, so duplicate negatives are expected and no
uniqueness guarantee is made. Zero-weight items are never drawn. Integer
weights avoid cumulative floating-point drift but limit the accepted weight
representation, and exact seeded output is tied to the tested Python random
generator. This function does not fit popularity, retry failed draws, update
weights, secure the random stream, or coordinate sampling across processes.

## Related Snippets

<!-- catalog:related:start -->
- [Sample a Weighted Stream with a Fixed-Size Reservoir](../data-processing/sample-a-weighted-stream-with-a-fixed-size-reservoir.md)
- [Sample Stream Items Independently with a Fixed Probability](../data-processing/sample-stream-items-independently-with-a-fixed-probability.md)
- [Fit and Apply an Exact Categorical Frequency Encoder](fit-and-apply-an-exact-categorical-frequency-encoder.md)
<!-- catalog:related:end -->
