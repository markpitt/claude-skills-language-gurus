# Async

Use async for concurrency where I/O latency dominates, with clear shutdown and cancellation behavior.

## Runtime Discipline

- Keep async tasks non-blocking; move CPU/blocking work to `tokio::task::spawn_blocking`.
- Avoid hidden blocking calls inside async contexts.

## Concurrency Patterns

- Use `tokio::spawn` for independent tasks and manage `JoinHandle`s.
- Use `tokio::select!` for cancellation races and shutdown coordination.
- Use `tokio::time::timeout` for bounded operations.
- Prefer bounded channels for backpressure in producer/consumer flows.

## Cancellation and Shutdown

- Design cancellation as a first-class behavior (timeouts, cancellation tokens, shutdown signals).
- Ensure spawned tasks are joined, aborted intentionally, or supervised.
- Avoid leaking detached background tasks.

## Common Mistakes

- Blocking runtime worker threads with CPU-heavy loops.
- Ignoring `JoinHandle` errors/panics.
- Using unbounded queues for bursty workloads without memory controls.
