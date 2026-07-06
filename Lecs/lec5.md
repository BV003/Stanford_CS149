# Work Distribution and Scheduling

TIP #1: Always implement the simplest solution first, then measure performance to determine if you need to do better.

## Three Major Work Assignment Strategies (Workload Balancing)

Static Assignment

Semi-Static Assignment
Dynamic Assignment

## Parallel Programming Common Patterns
Data Parallelism

Explicit Thread Management

Fork-Join Parallelism

## Cilk Plus Runtime（运行时系统） & Work-Stealing（工作窃取） Scheduler Deep Dive

## Critical Performance Pitfalls（性能陷阱） & Mitigations（对策）


Single global work queue bottleneck: High lock contention; solved via per-thread distributed deques + work stealing.
Overly fine-grained tasks: Serialization at critical lock sections; solved by batching to increase task granularity.
Late long-running tasks: Creates load imbalance; solved by prioritizing heavy tasks for early scheduling.
Excessive thread creation via raw pthreads: Heavy OS context switching, poor cache locality; solved by fixed-size worker thread pools.
Spawning tiny subproblems: Spawn overhead dominates parallel speedup; solved with parallel cutoff thresholds (serial fallback for small work units).

##  High-Level Design Takeaways（要点） for Parallel System Builders

Leverage static knowledge of workload first to minimize dynamic scheduling overhead; dynamic/work stealing is a fallback for unpredictable work.

Good parallel runtimes separate parallel abstraction (cilk_spawn/cilk_sync) from low-level scheduling implementation—programmers express potential parallelism without managing threads manually.

Locality-aware scheduling (work stealing from deque head/tail) balances load and preserves CPU cache performance.

Greedy join scheduling eliminates wasted idle time at synchronization barriers, a major source of lost parallel throughput.

Parallel slack (more tasks than cores) is necessary for load balance, but task granularity must be tuned to avoid synchronization overhead.