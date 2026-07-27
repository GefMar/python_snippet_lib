# Concurrency and Lifecycle

Scheduling, synchronization, cancellation, task ownership or
thread/process/resource lifecycle is the primary invariant.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Collect Thread-Pool Results and Errors as Futures Complete](collect-thread-pool-results-and-errors-as-futures-complete.md) | recipe | concurrency-control, resource-management | Run independent blocking calls in a thread pool while retaining both successful values and item-specific failures as each future finishes. |
| [Drain Bounded Deferred Writes Outside the Queue Lock](drain-bounded-deferred-writes-outside-the-queue-lock.md) | pattern | concurrency-control, networking, resource-management | Queue response bytes under a short lock, then perform potentially blocking writes without holding that queue lock. |
| [Gather Async Results with Bounded Concurrency](gather-async-results-with-bounded-concurrency.md) | recipe | concurrency-control, resource-management | Apply an asynchronous worker to a finite iterable while limiting active calls and preserving input order in the returned results. |
| [Guard an Async Resource with Explicit Lifecycle States](guard-an-async-resource-with-explicit-lifecycle-states.md) | pattern | concurrency-control, lifecycle-management, resource-management | Serialize asynchronous start and stop transitions so callers can observe and enforce one explicit resource lifecycle. |
| [Stop a Polling Worker Cooperatively with an Event](stop-a-polling-worker-cooperatively-with-an-event.md) | pattern | concurrency-control, resource-management | Run a bounded polling operation until another thread requests shutdown through a shared event that also interrupts idle waiting. |
| [Submit a Callable with a Snapshot of the Current Context](submit-a-callable-with-a-snapshot-of-the-current-context.md) | recipe | concurrency-control, interoperability, observability | Capture the current context for each thread-pool submission so the worker sees the caller's context variables without custom globals or exception wrappers. |
<!-- catalog:category:end -->
