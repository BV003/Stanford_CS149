# Lecture 2: Basic Architecture

This lecture covers the fundamental hardware architectures that support parallel processing, how parallel programs map to execution hardware, and how processors hide the latency of memory access.

---

## 1. Three Forms of Hardware Parallelism

Modern processors exploit three distinct paradigms of parallel execution to achieve high performance:

| Parallel Style | Discovery Mechanism | Control Overhead | Unit of Execution | Description |
| :--- | :--- | :--- | :--- | :--- |
| **Superscalar (ILP)** | **Hardware** | High | Single instruction stream | Automatically schedules independent instructions from a single thread stream across multiple execution units per clock. |
| **SIMD (Vector Processing)** | **Compiler / Hardware** | Low | Multiple data lanes, single instruction | Shares a single fetch/decode unit across multiple ALUs. Executes the same instruction on vectors of data. |
| **Multi-Core (TLP)** | **Software / Developer** | Medium to High | Independent instruction streams | Simultaneously executes completely different instruction streams on independent physical cores.(实现物理意义上的真并行) |

---

## 2. Expressing Parallelism in Code

To exploit these hardware paradigms, software must expose parallelism in different ways:

Process（操作系统中的基本单位）>Thread
### A. Thread（线程）-Level Parallelism (MIMD / Multi-core)
Developers partition work and spawn independent execution paths using APIs like C++ `std::thread`, POSIX threads (`pthreads`), or higher-level libraries.
* **Example (C++ Threads):**
  ```cpp
  // Splitting an array calculation in half across two threads
  std::thread my_thread(my_thread_func, &args);
  sinx(N - args.N, terms, x + args.N, y + args.N); // Work on main thread
  my_thread.join(); // Sync
  ```

### B. Data-Parallel Expression（表达）
Loop iterations are declared to be completely independent of one another.
* **Example (Fictitious `forall` construct):**
  ```cpp
  forall (int i from 0 to N) {
      y[i] = sin(x[i]);
  }
  ```
  This indicates to the compiler/runtime that loop iterations can be executed concurrently in any order, allowing automatic mapping to SIMD units or multi-core threads.

---

## 3. SIMD (Single Instruction, Multiple Data)（单指令多数据）

### Core Concept: Amortizing（摊销） Control
Instead of replicating the complex hardware required to fetch, decode, and manage instruction streams (control logic) for every arithmetic logic unit (ALU)（算术逻辑单元）, SIMD amortizes these control costs. A single fetch/decode unit broadcasts the same instruction to multiple ALUs operating in lockstep.

```
       [ Fetch / Decode ] (Control unit)
               │
      ┌────────┼────────┬────────┐
      ▼        ▼        ▼        ▼
   [ALU 0]  [ALU 1]  [ALU 2]  [ALU 3]  <── Lockstep Execution
```

### Explicit SIMD: AVX Intrinsics
C/C++ developers can write explicit（显式） vector code using intrinsics（内在） mapped to hardware vector registers (e.g., `__m256` which holds eight 32-bit float values).
* **Example (AVX2 Intrinsics):**
  ```cpp
  #include <immintrin.h>
  void sinx_vector(int N, float* x, float* y) {
      for (int i = 0; i < N; i += 8) {
          __m256 origx = _mm256_load_ps(&x[i]); // Loads 8 floats
          __m256 value = _mm256_mul_ps(origx, origx); // Multiplies element-wise（并行乘法（Multiply，硬件发出一条乘法指令，让寄存器中的 8 个通道同时进行各自的乘法计算）
          _mm256_store_ps(&y[i], value); // Stores 8 floats
      }
  }
  ```

---

## 4. Conditional SIMD Execution and Branch Divergence（分歧）

When a data-parallel loop contains conditional branches (`if/else`), the SIMD hardware must evaluate both paths because different data lanes may evaluate the condition differently. This is done via **execution masking**.（执行掩码）

### The Mechanism
1. The hardware computes a boolean predicate for all lanes（通道） (e.g., `t > 0.0`).
2. An **execution mask** (bit vector) is created (e.g., `1 1 1 0 0 1 0 0`).
3. During the `if` block, only ALUs with their mask bit set to `1` write out their results. Others are deactivated.
4. The mask is inverted (`0 0 0 1 1 0 1 1`), and the `else` block is executed.

```
Lane:       1   2   3   4   5   6   7   8
Condition:  T   T   T   F   F   T   F   F

Step 1 (If):
Mask:       1   1   1   0   0   1   0   0
ALU state: Active ──────────────────>   [Idle]

Step 2 (Else):
Mask:       0   0   0   1   1   0   1   1
ALU state:  [Idle] <─────────────────   Active
```

### Performance Impact of Divergence
* **Coherent（相干） Control Flow:** All lanes agree on the branch direction. Peak arithmetic throughput is maintained.
* **Divergent Control Flow:** Lanes disagree. The hardware serializes the branch execution paths.
* **Worst-Case Scenario:** Only 1 out of $W$ lanes is active at a time (where $W$ is the vector width). Arithmetic throughput drops to **$1/W$** (e.g., $12.5\%$ on 8-wide AVX2).（避免分支发散式性能优化中的第一铁律）

---

## 5. Memory Latency and Stalls

A processor **stalls** when it cannot execute the next instruction because a required resource (like data from a `load` instruction) is not yet available. Accessing main memory (DRAM) is a major source of stalls, taking **100s of clock cycles**.(访问DRAM主内存的速度要100个时钟周期，称其为内存墙)

```
ld r0, mem[r2]       <── Initiates load (takes ~100ns / ~200 cycles)
ld r1, mem[r3]
add r0, r0, r1       <── STALL! Cannot proceed until r0 and r1 are loaded
```

To combat stalls, modern systems use three primary latency-hiding mechanisms:

1. **Caches:** Small, extremely fast local SRAM memories that store recently or spatially close data.
2. **Prefetching:** Dedicated hardware analyzing access patterns (e.g., linear strides) and proactively fetching data into cache before the program explicitly requests it.
3. **Hardware Multi-Threading:** Storing multiple execution contexts (Register Files, Program Counters) on-chip. When one thread stalls, the core context-switches instantly (often in a single cycle) to another runnable thread.

---

## 6. Mathematical Analysis of Hardware Threading (Core Utilization)（核心利用率）

To understand core utilization under hardware multi-threading, we analyze the interplay between arithmetic execution time and memory latency.

### The Model
* Let $T_{calc}$ be the number of clock cycles a thread spends performing arithmetic before initiating a memory request.
* Let $L$ be the memory access latency (in clock cycles).
* Let $N_{threads}$ be the number of hardware-supported thread contexts.

### Single-Threaded Core Utilization
If only one thread is active, it runs for $T_{calc}$ cycles and then stalls for $L$ cycles.
$$\text{Core Utilization} = \frac{T_{calc}}{T_{calc} + L}$$

* **Example:** A thread performs $3$ arithmetic instructions and then loads from memory ($12$-cycle latency).
  $$\text{Utilization} = \frac{3}{3 + 12} = \frac{3}{15} = 20\%$$

### Multi-Threaded Core Utilization
With $N_{threads}$ active, the core schedules instructions from other threads during the stall cycles of the active thread.

To achieve **$100\%$ core utilization**, we must have enough threads to completely cover the memory latency $L$. The number of additional threads needed to run during one thread's stall is:
$$\text{Additional Threads} = \frac{L}{T_{calc}}$$

Therefore, the minimum total number of threads needed for $100\%$ utilization is:
$$N_{min\_threads} = 1 + \frac{L}{T_{calc}}$$

* **Using our example ($T_{calc} = 3$, $L = 12$):**
  $$N_{min\_threads} = 1 + \frac{12}{3} = 5 \text{ threads}$$
* If we have fewer than 5 threads (e.g., 2 threads), utilization increases proportionally but doesn't reach 100%:
  $$\text{Utilization} = \frac{N_{threads} \times T_{calc}}{T_{calc} + L} = \frac{2 \times 3}{15} = 40\%$$
* If we have more than 5 threads (e.g., 6 threads), utilization remains capped at $100\%$, and additional threads provide no further utilization benefits (though they may help with throughput buffer or memory scheduling).

### Key Takeaways
1. Multi-threading **does not reduce memory latency**. It only **hides** it by executing arithmetic instructions from other threads.
2. Programs with high **arithmetic intensity** (more calculations per memory load, higher $T_{calc}$) require **fewer threads** to hide the same memory latency.

---

## 7. Latency-Optimized (CPUs) vs. Throughput-Optimized (GPUs)

Modern CPU and GPU architectures are optimized for fundamentally different performance goals:

```
                  CPU                                         GPU
┌──────────────────────────────────────┐   ┌──────────────────────────────────────┐
│  [Branch Predictor]   [Large Cache]  │   │  [Core] [Core] [Core] [Core] [Core]  │
│                                      │   │  [Core] [Core] [Core] [Core] [Core]  │
│  [Execution Core]    [Control Logic] │   │  [Core] [Core] [Core] [Core] [Core]  │
│  (Sophisticated Out-of-Order Engine) │   │  (Hundreds of simple SIMD ALUs with  │
│  - Statically hides latency          │   │   massive multi-threading)           │
└──────────────────────────────────────┘   └──────────────────────────────────────┘
```

* **CPU Philosophy (Latency-Oriented):** 
  * Optimizes the execution speed of a *single instruction stream*.
  * Spends significant chip area on large caches (to minimize memory latency), sophisticated branch prediction (to avoid instruction stalls), and complex out-of-order execution pipelines.
  * Supported hardware thread contexts per core are low (usually $2$, e.g., Intel Hyper-Threading).
* **GPU Philosophy (Throughput-Oriented):**
  * Optimizes the *total volume of work completed per unit time*.
  * Allocates the majority of silicon area to arithmetic execution units (ALUs).
  * Uses very simple caches and minimal branch prediction.
  * Relies on massive hardware multi-threading (thousands of active contexts) to dynamically hide memory latency.

---

## 8. Summary of Key Terminology

* **Instruction Stream:** A sequence of instructions executed by a logical thread of control.
* **SIMD Execution:** Running a single instruction over multiple data elements simultaneously across independent ALU pipelines.
* **Coherent Control Flow:** All concurrent execution paths within a SIMD vector execute the same instruction path (no branch divergence).
* **Hardware Multi-threading:** Processors holding multiple register files and instruction pointers on-chip to interleaving execution of different threads without OS-level context switch overhead.
  * **Interleaved Multi-threading:** The core switches threads on every clock cycle (e.g., round-robin or based on stalls).
  * **Simultaneous Multi-threading (SMT):** The core can issue instructions from multiple threads within the *same* clock cycle to maximize execution unit utilization.
