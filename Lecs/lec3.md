# Lecture 3: Performance Terminology, Pipelining, and ISPC (SPMD)

NVIDIA V100

计算需求：要让所有 ALU 满载，理论上需要接近 56 TB/s 的数据传输速度。
物理现实（V100 的 HBM2 显存）：V100 采用了当时最先进的高带宽显存（HBM2），其物理带宽上限约为 900 GB/s（即 0.9 TB/s）。

Throughput

Memory bandwidth

The rate at which the memory system can provide data to a processor

Overcoming bandwidth limits is often the most important
challenge facing software developers targeting modern
throughput-optimized systems.

SPMD（单程序多数据）

“Gang”（线程群）