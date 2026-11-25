# Performance

Optimize with data, not intuition.

## Measurement First

- Profile before and after changes.
- Use representative workloads and realistic data sizes.
- Track throughput, tail latency, and memory where relevant.

## Common High-Value Improvements

- Preallocate where sizes are known (`with_capacity`).
- Reuse buffers in hot loops.
- Avoid unnecessary clones and temporary allocations.
- Prefer slices/iterators over collecting intermediates unless readability suffers.

## Async/IO Performance

- Prevent blocking on async runtime workers.
- Bound queues to preserve backpressure.
- Batch I/O where protocol semantics allow.

## Keep It Simple

- Choose simple improvements with clear wins first.
- Introduce unsafe or complex specialization only with strong benchmark evidence.
