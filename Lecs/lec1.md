# Lecture 1: Why Parallelism? Why Efficiency?

This lecture serves as the introduction to parallel computing, establishing the physical and historical context that forced the computing industry to transition from single-core performance scaling to parallel and specialized hardware.

---

## 1. The Transition to Parallelism (The Death of the "Free Lunch")

For decades, software developers enjoyed a "free lunch": any program written for a single-threaded processor would automatically run faster on next year's processor without modifying a single line of code. This was driven by two main mechanisms:

1. **Instruction-Level Parallelism (ILP):** Hardware architects designed processors that could automatically find independent instructions within a single instruction stream and execute them in parallel (using pipelining, superscalar execution, out-of-order execution, etc.).
2. **Frequency Scaling:** Chip manufacturing allowed clock frequencies to increase rapidly, executing more instructions per second.

```
+-------------------------------------------------------------+
|                     The Pre-2005 Era                        |
|                                                             |
|  [Single Threaded Code] ──> [Faster Single-Core Processor]  |
|                                 (Automatic Speedup!)        |
+-------------------------------------------------------------+
```

### The Transition (Circa 2005)
This free lunch ended because single-thread performance scaling slowed down to near-zero. Two physical walls were hit:
* **The ILP Wall (Diminishing Returns):** Most available ILP is fully exploited by a processor that can issue 4 instructions per clock. Attempting to build processors capable of issuing more instructions yields minimal performance benefit for massive increases in transistor cost and complexity.
* **The Power Wall:** Transistor density continued to increase (Moore's Law), but clock frequency stopped increasing due to heat dissipation limits.

```
+-------------------------------------------------------------+
|                     The Post-2005 Era                       |
|                                                             |
|  [Parallel / Threaded Code] ──> [Multi-Core Processors]     |
|                                   (Requires Code Rewrites)  |
+-------------------------------------------------------------+
```

---

## 2. The Power Wall and the Physics of Transistors

Power consumption is the dominant physical design constraint in modern processors. High power results in high heat, which requires expensive cooling systems or leads to thermal throttling (clocking down a chip to prevent physical damage).

### Power Consumption Formula
The dynamic power consumed by a processor's transistors is modeled by:

$$\text{Power}_{\text{dynamic}} \propto C \times V^2 \times f$$

Where:
* $C$ = Capacitive load (proportional to transistor count and chip layout)
* $V$ = Operating voltage
* $f$ = Clock frequency

### Implications
* Operating voltage $V$ is tied to frequency $f$. To run at a higher frequency, you must supply a higher voltage.
* Because voltage is squared ($V^2$), increasing frequency leads to a cubic or super-linear increase in power consumption.
* Conversely, lowering frequency slightly allows a much larger drop in voltage, significantly reducing power and heat. 

---

## 3. The Three Course Themes of CS149

The course is structured around three fundamental themes:

### Theme 1: Designing and Writing Parallel Programs that Scale
Parallel programming requires a shift in how you think about algorithms. It involves:
1. **Decomposition:** Breaking the computation into smaller, independent pieces of work.
2. **Assignment:** Mapping those pieces of work to available processor cores.
3. **Orchestration:** Managing communication and synchronization between cores to minimize coordination overhead.

### Theme 2: Understanding Parallel Hardware Implementations
To write efficient parallel code, you must understand how the hardware behaves. Different machine architectures have vastly different performance characteristics and cost/convenience trade-offs.

### Theme 3: Thinking About Efficiency
**Fast is not the same as efficient.** 
* A program may run faster simply because it has access to more hardware, but it might be utilizing those resources extremely poorly.
* For example, obtaining a $2\times$ speedup on a system with $10$ physical cores means $80\%$ of your processing capability is wasted.
* High efficiency maximizes the amount of useful computation achieved per unit of energy, silicon area, or dollar.

---

## 4. The Data Movement and Energy Bottleneck

Modern processing elements (ALUs) are incredibly fast and energy-efficient. The primary bottleneck to both performance and power efficiency is **data movement**.

### Energy Cost of Operations (Ballpark Numbers)
The energy cost of running an operation is dominated by fetching the inputs from the memory hierarchy, rather than performing the math itself:

| Operation | Energy Cost (approx.) | Relative Cost |
| :--- | :---: | :---: |
| Integer Addition (32-bit) | **$1\text{ pJ}$** | $1\times$ |
| Floating Point Addition (32-bit) | **$20\text{ pJ}$** | $20\times$ |
| Reading 64 bits from local SRAM (1mm away) | **$26\text{ pJ}$** | $26\times$ |
| Reading 64 bits from off-chip DRAM | **$1200\text{ pJ}$** | $1200\times$ |

### Implications
* Moving data from off-chip DRAM to a CPU/GPU core costs **over $1000\times$** more energy than performing a mathematical operation.
* A mobile system reading $10\text{ GB/sec}$ from memory consumes roughly $1.6\text{ watts}$—which already exceeds the entire thermal/power budget of a typical smartphone processor ($1\text{ to }2\text{ watts}$).
* **Exploiting locality (reusing data within registers and local caches) is the single most important technique for writing energy-efficient and high-performance parallel code.**

---

## 5. Summary Checklist

* **Why did single-core scaling stop?** Tapped out Instruction-Level Parallelism (ILP) and thermal limits (The Power Wall).
* **What is the goal of parallel programming?** To decompose, assign, and coordinate work to scale performance across multiple cores.
* **Why does data access matter so much?** Data movement consumes orders of magnitude more energy than arithmetic, making locality the primary key to software efficiency.
