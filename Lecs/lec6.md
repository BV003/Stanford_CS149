# Lecture 6: Performance Optimization Part II — Locality, Communication, and Contention


---

## 1. Shared Address Space Model

- All threads read/write variables in a single shared address space
- Hardware must implement this abstraction efficiently (caches, interconnects)
- Real implementations are complex: L1/L2/L3 caches, ring interconnect, crossbar, NUMA

**Key architectures:**
- Intel Core i7 (Kaby Lake): ring interconnect with L3 cache slices
- SUN Niagara 2: crossbar switch connecting 8 cores
- NUMA: memory access latency differs by which core you're on (multi-socket systems)

---

## 2. Message Passing Model

- Each thread has its own private address space
- Communication via explicit `send()` / `recv()` calls
- Hardware only needs to pass messages between nodes (no shared memory HW required)
- Used in clusters and supercomputers (Infiniband, etc.)

**Synchronous(同步) (blocking) send/recv:**
- `send()` returns only when receiver acknowledges data is in its address space
- `recv()` returns only when data is copied and ack sent back
- ⚠️ Deadlock risk — order matters (fix with alternating send/recv patterns)
两边都在等对方先收消息，永远等不到 ACK，双方永久阻塞 → 死锁。如果是异步 / 非阻塞 send，发送完立刻返回，不会卡住程序，就不会死锁。

**Asynchronous (non-blocking) send/recv:**
- `send()`/`recv()` return immediately with a handle
- Use `checksend()`/`checkrecv()` to check completion
- Enables overlapping（重叠） computation with communication

---

## 3. Communication = Extended Memory Hierarchy（等级）

Think of all communication uniformly:

| Level | Latency | Bandwidth |
|---|---|---|
| Registers | Lowest | Highest |
| L1 Cache | ↓ | ↑ |
| L2 Cache | | |
| L3 Cache | | |
| Local Memory | | |
| Remote Memory (1 hop) | ↓ | ↓ |
| Remote Memory (N hops) | Highest | Lowest |

---

## 4. Arithmetic Intensity（算术强度）

```
Arithmetic Intensity = amount of computation / amount of communication
```

- Higher is better
- `1 / arithmetic intensity` = communication-to-computation ratio
- Modern processors have much more compute than bandwidth → need high arithmetic intensity（我们又遇见了内存墙）

---

## 5. Inherent（固有） vs. Artifactual（人为） Communication

**Inherent communication:** fundamental data movement required by the algorithm

- Reduce via better assignment:
  - 1D blocked → `AI ∝ N/P`
  - 2D blocked → `AI ∝ N/√P` (better scaling!)

**Artifactual communication:** extra communication caused by system implementation（we can eliminate this.）

- Cache line granularity（粒度） (load 64B when you need 4B)
- Capacity misses (cache too small to retain data between accesses)
- Unnecessary loads (load then overwrite entire cache line)

---

## 6. Techniques to Reduce Communication

### Blocking (tiling)（平铺）
- Reorder computation to reuse data while it's still in cache
- Grid solver: process in blocks instead of row-by-row
- Result: fewer cache misses per output element

### Loop Fusion（环融合）
- Merge separate loops into one to avoid storing/loading intermediate（中间的） arrays
- Example: `E = D + (A+B)*C` in one loop instead of three separate loops
- Higher arithmetic intensity (3 math ops per 5 memory ops vs 3/9)

### Data Sharing
- Schedule threads working on the same data on the same processor

---

## 7. Contention（争抢）

- A shared resource can only handle limited throughput
- Many simultaneous（同时的） requests → contention → higher latency
- Examples: shared work queue, shared counter, shared memory bus

**Solutions:**
- Tree-structured communication (reduce contention, increase latency)(防止网卡等待，确保每一个时刻都只有一个网卡通信)
- Distributed work queues (each thread has own queue, steal when empty)
- Stagger（错峰） access to shared resources
- Replicate contended resources

---

## 8. Performance Analysis Tools

### Roofline Model
- X-axis: arithmetic intensity
- Y-axis: max attainable（可实现的） throughput
- Horizontal region → compute-bound
- Diagonal region → memory-bandwidth-bound

### High Watermark(高水位) Experiments
- **Add math ops**: does time increase linearly? → compute-bound
- **Change all array accesses to `A[0]`**: how much faster? → upper bound on locality improvement
- **Remove all atomics/locks**: upper bound on sync overhead
- **Remove math, keep same loads**: if time barely drops → memory-bound

### Hardware Performance Counters
- Intel PCM, VTune, PAPI, oprofile
- Count: instructions, cycles, cache hits/misses, bytes read from memory

---

## 9. Scaling Pitfalls（陷阱）

| Problem Size | Machine Size | Issue |
|---|---|---|
| Too small | Large | Communication dominates, no speedup or slowdown |
| Small | Small | May fit in cache → super-linear speedup |
| Too large | Small | Working set exceeds RAM → disk thrashing |

- **Fixed problem size scaling** can mislead
- Prefer scaling problem size with machine size (weak scaling)
- Always compare against **best sequential program**, not parallel-on-1-core

---

## Summary Tips

1. **Measure first** — always try simplest parallel solution, then profile
2. **Establish high watermarks** — are you compute, bandwidth, or sync bound?
3. **Watch problem size** — is it well-sized for the machine?
4. **Reduce communication** — blocking, fusion, better assignment
5. **Reduce contention** — distribute work, replicate resources
6. **Overlap communication and computation** — use async operations
