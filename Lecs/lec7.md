# Lecture 7: GPU Architecture & CUDA Programming


---

## 1. What GPUs Were Originally Designed For: 3D Rendering

- Input: 3D scene (triangle meshes, materials, lights, camera)
- Output: image (pixels)
- Core workload: for each pixel covered by a triangle, compute surface color
- **Shader programs**: pure functions run once per pixel/fragment, massively parallel

**Shader example (GLSL):**
```
void myShader() {
  vec3 kd = texture2D(myTexture, uv);           // sample texture
  kd *= clamp(dot(lightDir, norm), 0.0, 1.0);   // simple lighting
  return vec4(kd, 1.0);
}
```

---

## 2. GPGPU Evolution (2002–2007)

| Year | Milestone |
|---|---|
| 2002-03 | Early hacks: render two triangles covering screen, shader = compute kernel |
| 2004 | **Brook** language (Stanford): abstract GPU as data-parallel processor |
| 2007 | **NVIDIA Tesla** + **CUDA**: first non-graphics "compute mode" GPU interface |

- Before 2007: only way to program GPU was through graphics pipeline (`drawPrimitives`)
- After 2007: simply `launch(myKernel, N)` — run N instances of a kernel

---

## 3. CUDA Programming Model

### Execution hierarchy
```
Grid → Thread Blocks → CUDA Threads (up to 3D indexing)
```

- **Host** (CPU): serial code, launches kernels
- **Device** (GPU): SPMD execution of `__global__` kernel functions
- `__device__` functions: called from device code only

### Key example — matrix addition:
```cuda
// Host launch
dim3 threadsPerBlock(4, 3);
dim3 numBlocks(Nx/4, Ny/3);
matrixAdd<<<numBlocks, threadsPerBlock>>>(A, B, C);

// Device kernel
__global__ void matrixAdd(float A[Ny][Nx], float B[Ny][Nx], float C[Ny][Nx]) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    int j = blockIdx.y * blockDim.y + threadIdx.y;
    C[j][i] = A[j][i] + B[j][i];
}
```

- Number of threads is **explicit** in the launch (not determined by data size)
- Guard out-of-bounds when data size isn't a multiple of block size

---

## 4. CUDA Memory Model

Three device address spaces:

| Memory Type | Scope | Speed | Size |
|---|---|---|---|
| **Per-thread private** | One thread | Fastest | Small |
| **Per-block shared** (`__shared__`) | All threads in block | Fast (on-chip SRAM) | ~48-128 KB |
| **Device global** | All threads, host | Slow (DRAM) | GBs |

- Host ↔ Device: `cudaMemcpy(deviceA, A, bytes, cudaMemcpyHostToDevice)`
- Host and device have **separate address spaces**

---

## 5. 1D Convolution Example — Two Versions

### Version 1: Global memory only
```cuda
#define THREADS_PER_BLK 128
__global__ void convolve(int N, float* input, float* output) {
    int index = blockIdx.x * blockDim.x + threadIdx.x;
    float result = 0.0f;
    for (int i=0; i<3; i++)
        result += input[index + i];    // each thread loads from global memory
    output[index] = result / 3.f;
}
```
→ 3 × 128 = 384 global memory loads per block

### Version 2: Use shared memory
```cuda
__global__ void convolve(int N, float* input, float* output) {
    __shared__ float support[THREADS_PER_BLK+2];
    int index = blockIdx.x * blockDim.x + threadIdx.x;
    support[threadIdx.x] = input[index];
    if (threadIdx.x < 2)
        support[THREADS_PER_BLK + threadIdx.x] = input[index + THREADS_PER_BLK];
    __syncthreads();           // barrier!
    float result = 0.0f;
    for (int i=0; i<3; i++)
        result += support[threadIdx.x + i];
    output[index] = result / 3.f;
}
```
→ Only 130 loads per block (cooperative load + reuse via shared memory)

---

## 6. Synchronization in CUDA

- `__syncthreads()` — barrier within a thread block
- **Atomic operations**: `atomicAdd`, `atomicInc`, etc. (on global + shared memory)
- **Implicit barrier** at kernel return (all blocks finish)

---

## 7. GPU Architecture: NVIDIA V100 Deep Dive

### Streaming Multiprocessor (SM) — key specs
- 80 SMs per V100 chip
- 4 sub-cores per SM
- Each sub-core: 16 fp32 ALUs, 16 int ALUs, 8 fp64 ALUs, load/store units, tensor cores
- Shared memory + L1: 128 KB per SM

### Warp = 32 threads
- Threads 0–31 in a block = one warp (32–63 = next warp, etc.)
- Threads in a warp execute in **SIMT** fashion (Single Instruction, Multiple Threads)
- If all 32 threads share the same instruction → SIMD execution across 16 ALUs (2 clocks)
- If threads diverge → performance penalty (masked execution)

### V100 key numbers
| Metric | Value |
|---|---|
| Clock | 1.245 GHz |
| SM count | 80 |
| fp32 ALUs | 5120 |
| Peak fp32 | ~12.7 TFLOPs |
| Max warps/chip | 5120 (163,840 CUDA threads) |
| HBM2 bandwidth | ~900 GB/sec |

---

## 8. How a CUDA Kernel Runs on GPU

1. **Host sends kernel launch** command to GPU
2. **Thread block scheduler** maps blocks to SMs
   - Respects resource limits: threads, shared memory, registers per SM
   - Blocks are scheduled independently, **in any order**
3. **Within a block**: all threads run concurrently on **same SM**
   - Enables fast shared memory communication
   - Supports `__syncthreads()`
4. **When block completes**: resources freed, scheduler assigns next block

---

## 9. Key Design Insights

| Concept | CUDA Mapping |
|---|---|
| Data-parallel decomposition | Thread blocks (machine-independent count) |
| SPMD within a block | All threads run same kernel, different data |
| Work scheduling | Blocks → SMs (GPU HW scheduler, like ISPC tasks) |
| SIMD execution | Warp of 32 threads (like ISPC gang, but dynamic) |
| Address space separation | Host memory ↔ Global ↔ Shared ↔ Per-thread |

### CUDA Thread vs pthread
- A CUDA thread is **lighter weight** than a pthread
- GPU hardware manages thousands of threads efficiently via warps
- No per-thread OS scheduling overhead

---

## 10. Bonus: Persistent Threads Style

- Launch exactly as many blocks as can fill the GPU
- Each block runs a `while(1)` loop with `atomicInc` to grab work
- Circumvents GPU scheduler — programmer manages work assignment
- Requires knowledge of GPU hardware specifics (fragile across GPUs)

---

## 11. Summary Cheat Sheet

```
Programming model:
  Data-parallel decomposition → independent thread blocks
  Inside block → SPMD + shared address space + barriers + atomics

Memory hierarchy:
  Global (slow, GBs) → Shared (fast, KBs) → Per-thread (fastest)

Hardware mapping:
  Block → SM (same SM for all threads in block)
  32 threads → Warp → SIMT execution on SIMD ALUs

Performance rule:
  When threads share data → use __shared__ memory + __syncthreads()
```
