# Concurrency and Lifecycle

Scheduling, synchronization, cancellation, task ownership or
thread/process/resource lifecycle is the primary invariant.

## Snippets

<!-- catalog:category:start -->
| Snippet | Type | Use Cases | Problem |
| --- | --- | --- | --- |
| [Collect Thread-Pool Results and Errors as Futures Complete](collect-thread-pool-results-and-errors-as-futures-complete.md) | recipe | concurrency-control, resource-management | Run independent blocking calls in a thread pool while retaining both successful values and item-specific failures as each future finishes. |
| [Gather Async Results with Bounded Concurrency](gather-async-results-with-bounded-concurrency.md) | recipe | concurrency-control, resource-management | Apply an asynchronous worker to a finite iterable while limiting active calls and preserving input order in the returned results. |
| [Stop a Polling Worker Cooperatively with an Event](stop-a-polling-worker-cooperatively-with-an-event.md) | pattern | concurrency-control, resource-management | Run a bounded polling operation until another thread requests shutdown through a shared event that also interrupts idle waiting. |
<!-- catalog:category:end -->
