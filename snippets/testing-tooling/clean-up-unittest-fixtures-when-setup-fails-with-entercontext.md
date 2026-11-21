---
title: "Clean Up unittest Fixtures When setUp Fails with enterContext"
snippet_type: testing-technique
use_cases:
  - lifecycle-management
  - resource-management
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - ../concurrency-lifecycle/transfer-bounded-synchronous-cleanup-ownership-with-exitstack-pop-all.md
  - ../storage-databases/hand-off-bounded-bytes-through-a-scoped-named-temporary-path.md
---

# Clean Up unittest Fixtures When setUp Fails with enterContext

## Idea and Problem

Keep a context-managed fixture alive throughout a unittest case and release it even when later setup work raises.

A `with` block inside `setUp` releases the resource before the test can use
it. Storing the cleanup for `tearDown` leaves a gap because `unittest` skips
`tearDown` if `setUp` fails. `TestCase.enterContext` enters the resource and
immediately registers its exit method on the test's cleanup stack.

## When to Use

Use this when a synchronous context manager owns a per-test resource and the
test needs that resource through `tearDown`. Enter the resource before setup
steps that depend on it, such as preparing input files in a temporary
directory.

## Implementation

```python
from pathlib import Path
from tempfile import TemporaryDirectory
from unittest import TestCase


class ExportFixtureCase(TestCase):
    def setUp(self) -> None:
        self.stages = ["setup"]
        self.workspace = Path(self.enterContext(TemporaryDirectory()))
        self.input_file = self.workspace / "rows.csv"
        self.prepare_input()

    def prepare_input(self) -> None:
        self.input_file.write_text("name,count\nblue,3\n", encoding="utf-8")

    def test_read_input(self) -> None:
        self.assertEqual(self.input_file.read_text(encoding="utf-8"), "name,count\nblue,3\n")
        self.stages.append("test")

    def tearDown(self) -> None:
        self.assertTrue(self.workspace.is_dir())
        self.stages.append("teardown")
```

## Example

```python
class BrokenExportFixtureCase(ExportFixtureCase):
    def prepare_input(self) -> None:
        raise OSError("fixture data could not be prepared")


working = ExportFixtureCase("test_read_input")
broken = BrokenExportFixtureCase("test_read_input")
working_result = working.run()
broken_result = broken.run()

assert working_result.wasSuccessful()
assert working.stages == ["setup", "test", "teardown"]
assert len(broken_result.errors) == 1 and broken.stages == ["setup"]
assert not working.workspace.exists() and not broken.workspace.exists()
```

## Trade-offs and Limitations

`enterContext` is available from Python 3.11. The test framework runs cleanup
in reverse registration order after `tearDown`, or directly after a failing
`setUp`. The cleanup is registered only after `__enter__` succeeds. A context
manager that partially acquires resources inside its own failing entry must
handle those resources itself.

The registered exit method receives no exception details from the test or its
setup. Use an ordinary `with` block around the operation when the exit method
needs to see an exception, for example to choose a transaction outcome.

## Related Snippets

<!-- catalog:related:start -->
- [Transfer Bounded Synchronous Cleanup Ownership with ExitStack Pop All](../concurrency-lifecycle/transfer-bounded-synchronous-cleanup-ownership-with-exitstack-pop-all.md)
- [Hand Off Bounded Bytes Through a Scoped Named Temporary Path](../storage-databases/hand-off-bounded-bytes-through-a-scoped-named-temporary-path.md)
<!-- catalog:related:end -->
