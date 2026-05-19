---
title: "11 - Performance Analysis"
created: 2026-05-19
updated: 2026-05-19
tags: [computer-architecture, performance, amdahl, gustafson, roofline, profiling, ipc, perf-counters]
aliases: []
---

# 11 - Performance Analysis

[toc]

> **TL;DR:** Performance analysis is the discipline of identifying why a program runs slower than possible and determining what bottleneck to fix. Amdahl's Law bounds the speedup from parallelism given a serial fraction; Gustafson's Law reframes this for workloads that scale with available resources. The roofline model identifies whether a kernel is compute-bound or memory-bandwidth-bound. Practical analysis uses hardware performance counters — IPC, cache miss rates, branch misprediction rates, stall cycles — available via Linux `perf`, Intel VTune, and NVIDIA Nsight Compute. Without measurement, optimisation is guessing.

## Vocabulary

**Speedup**: The ratio of the original execution time to the optimised execution time. Speedup > 1 means faster.

```math
\text{Speedup} = \frac{T_{original}}{T_{optimised}}
```

---

**Amdahl's Law**: The theoretical maximum speedup from parallelising a fraction (1−s) of a program, where s is the serial fraction.

```math
\text{Speedup}(N) = \frac{1}{s + \frac{1-s}{N}}
```

---

**Gustafson's Law**: The observation that the serial fraction s typically decreases as the problem size scales up. Large problems are often more parallelisable than small problems — the pessimistic fixed-serial-fraction assumption in Amdahl's Law does not hold for scaled workloads.

```math
\text{Scaled speedup}(N) = N - s \cdot (N - 1)
```

---

**IPC (Instructions Per Cycle)**: The number of instructions a CPU retires per clock cycle. Closely related to CPI (cycles per instruction = 1/IPC). IPC varies with instruction mix, cache behaviour, and branch prediction.

---

**Performance counter (PMU counter)**: A hardware register in the CPU that counts microarchitectural events: clock cycles, instructions retired, cache misses, branch mispredictions, stall cycles, etc. Readable via `perf`, `RDPMC`, or vendor profilers.

---

**Hot path**: The fraction of execution time spent in a small region of code (often 80% of time in 20% of the code, or stronger in practice). Profiling identifies hot paths.

---

**Sampling profiler**: A profiler that periodically interrupts the program and records the current PC (and optionally the call stack). Overhead: typically 1–5%. Examples: `perf record`, `gprof`, `Instruments.app`.

---

**Instrumentation profiler**: A profiler that modifies the code (at compile time or via binary rewriting) to count every function call or instruction. More precise but higher overhead. Examples: `gprof` call-graph mode, Intel PIN.

---

**Roofline model**: A visual model plotting achievable FLOPS against arithmetic intensity (FLOPs/byte). The horizontal line is peak compute; the sloped line is peak bandwidth × AI. The kernel's performance is bounded by whichever line it hits first.

---

**Arithmetic intensity (AI)**: FLOPs per byte of memory traffic. The dividing characteristic between compute-bound and memory-bound kernels.

```math
\text{AI} = \frac{\text{FLOPs performed}}{\text{bytes transferred (DRAM)}}
```

---

**FLOPS utilisation**: Achieved FLOPS / peak FLOPS. For a well-optimised matmul on H100: ~80–90%. For a random-access memory workload: <5%.

---

**Memory bandwidth utilisation**: Achieved GB/s / peak GB/s. For a streaming benchmark (STREAM): >90%. For a pointer-chasing linked list: <1%.

---

**Profiling granularity**: The finest resolution at which the profiler can attribute execution time: function, basic block, line, instruction.

---

## Intuition

Performance analysis is applied scientific method. You have a hypothesis ("this loop is slow because of cache misses"), you measure, you confirm or refute, you fix, you measure again. Skipping the measurement and going straight to "optimising" based on intuition is cargo-cult engineering — it often wastes time on the wrong bottleneck.

Amdahl's Law is both a guide and a warning. It says: make the common case fast, because no amount of optimisation on the uncommon case can beat optimising the dominant fraction. If 90% of time is in a hot loop and 10% in I/O, making the I/O infinite-speed only gives 1.11× speedup. Fix the 90%.

The roofline model's genius is that it tells you what kind of optimisation to pursue before you write a single line of code. Compute-bound kernels need algorithmic improvements or better use of hardware FUs (Tensor Cores). Memory-bound kernels need better data layout, caching, or prefetching. The wrong optimisation — trying to reduce FLOPs on a memory-bound kernel — wastes effort entirely.

## Amdahl's Law

Amdahl's Law (1967) gives the maximum speedup achievable by parallelising a fraction of a program:

```math
\text{Speedup}(N) = \frac{1}{s + \frac{1-s}{N}}
```

where s is the serial fraction and N is the number of processors. As N → ∞:

```math
\lim_{N \to \infty} \text{Speedup}(N) = \frac{1}{s}
```

Even with infinite cores, the maximum speedup is bounded by 1/s.

**Concrete table:**

| Serial fraction s | Max speedup (N→∞) | Speedup at N=16 | Speedup at N=1024 |
| :---: | :---: | :---: | :---: |
| 0.5% | 200× | 14.9× | 180× |
| 5% | 20× | 10.6× | 19.6× |
| 10% | 10× | 6.4× | 9.8× |
| 50% | 2× | 1.9× | 2.0× |

> [!IMPORTANT]
> Amdahl's Law applies to *any* optimisation, not just parallelism. If 5% of execution time is in a function you can make infinitely fast, the program is at most 1.053× faster (Amdahl with s=0.95). Before optimising any specific component, measure its fraction of total execution time. Optimising a 1% bottleneck gives at most 1% improvement regardless of how clever the optimisation is.

### Gustafson's Law

Gustafson (1988) observed that Amdahl's analysis is pessimistic because it holds problem size fixed. In practice, when more computing resources are available, scientists and engineers increase the problem size proportionally (finer simulation grid, larger batch, longer sequence). The serial fraction s then typically decreases because the parallel portion grows while the serial setup/bookkeeping stays roughly constant:

```math
\text{Scaled speedup}(N) = N - s(N-1)
```

For s=0.05: scaled speedup(1024) = 1024 − 0.05 × 1023 ≈ **973×** — far better than Amdahl's 19.6×. The key insight: when you scale up the workload with the hardware, the parallel fraction dominates.

Both laws are correct for their respective assumptions. Amdahl describes fixed-workload scaling; Gustafson describes scaled-workload (weak scaling) scenarios. Real HPC and ML training typically operates under Gustafson conditions.

## The Roofline Model

The roofline model (Williams, Waterman, Patterson, 2009) plots achievable performance on the y-axis against arithmetic intensity on the x-axis, with two bounding lines:

1. **Compute ceiling:** Peak FLOPS (horizontal line). No kernel can exceed this regardless of AI.
2. **Bandwidth ceiling:** Peak bandwidth × AI (diagonal line). No kernel can exceed this for a given AI.

A kernel's achievable performance = min(peak FLOPS, bandwidth × AI).

```
FLOPS
  |
  |         compute bound region
  |            ________________ (peak FLOPS)
  |           /                
  |          /  (slope = peak bandwidth)
  |         /  memory-bandwidth bound
  |        /
  +--------+--------------------> AI (FLOPS/byte)
           ^ ridge point = peak FLOPS / peak bandwidth
```

**Ridge point (balance point):** AI at the ridge = Peak FLOPS / Peak Bandwidth. Operations with AI > ridge are compute-bound; AI < ridge are memory-bandwidth-bound.

For H100 SXM (BF16 Tensor Core): ridge = 2000 TFLOPS / 3.35 TB/s ≈ **597 FLOPS/byte**.

For an Apple M4 Pro (FP32): ridge = 14 TFLOPS / 273 GB/s ≈ **51 FLOPS/byte**.

The M4's lower ridge means more kernels become compute-bound on Apple Silicon — its unified memory architecture gives it much better bandwidth-per-TFLOP than discrete GPU systems.

## Hardware Performance Counters

Modern CPUs expose hundreds of hardware events via the PMU (Performance Monitoring Unit). These are the raw numbers underlying every performance claim.

### Key events and what they mean

| Counter | What it measures | Interpretation |
| :--- | :--- | :--- |
| `instructions` | Instructions retired | Instruction count in the Iron Law |
| `cycles` | Elapsed clock cycles | Denominator of IPC |
| `cache-misses` | L1/L2/L3 cache miss count | Main source of memory-bound stalls |
| `branch-misses` | Branch mispredictions | Main source of control hazard stalls |
| `stalled-cycles-frontend` | Cycles the fetch/decode stage is stalled | Instruction supply bottleneck |
| `stalled-cycles-backend` | Cycles the backend (execution) is stalled | Memory or compute bottleneck |
| `L1-dcache-load-misses` | L1 data cache miss count | First-level miss frequency |
| `LLC-load-misses` | Last-level cache (L3) misses to DRAM | DRAM traffic measure |

### Linux perf Usage

`perf stat` gives the summary statistics. `perf record` + `perf report` give function-level attribution. `perf annotate` gives instruction-level attribution.

```bash
# Basic IPC and miss rate for a program
perf stat -e cycles,instructions,cache-misses,branch-misses ./your_program

# Record a profile for flame graph
perf record -g ./your_program   # -g = with call graphs
perf report --stdio             # text summary
# or: perf script | flamegraph.pl > flame.svg  (needs FlameGraph scripts)

# Hardware events for cache analysis
perf stat -e L1-dcache-load-misses,LLC-load-misses,stalled-cycles-backend ./your_program
```

### Intel VTune / AMD uProf / Apple Instruments

Vendor profilers provide richer analysis: micro-architectural stall breakdown (where in the pipeline cycles are wasted), memory access patterns, lock contention, and NUMA analysis. For GPU profiling, use NVIDIA Nsight Compute (`ncu`) for kernel-level roofline analysis.

## Real-world Example

The following Python and C code demonstrates profiling a Python function, identifying the bottleneck, and fixing it — the full analysis loop.

```python
"""
Performance analysis workflow example.
Find the bottleneck in a text tokenisation function.
"""
import cProfile
import pstats
import io
import time

def tokenise_slow(text: str) -> list[str]:
    """Slow O(n^2) tokeniser using repeated string concatenation."""
    tokens = []
    current = ""
    for char in text:
        if char == " ":
            if current:
                tokens = tokens + [current]   # O(n) list copy on each append!
                current = ""
        else:
            current = current + char           # O(n) string concatenation!
    if current:
        tokens = tokens + [current]
    return tokens

def tokenise_fast(text: str) -> list[str]:
    """Fast O(n) tokeniser using list.append() and str.split()."""
    return text.split()

# Generate a large input
words = ["hello", "world", "this", "is", "a", "test"]
text = " ".join(words * 100000)   # 600,000 words

# Profile the slow version
profiler = cProfile.Profile()
profiler.enable()
result_slow = tokenise_slow(text)
profiler.disable()

stream = io.StringIO()
ps = pstats.Stats(profiler, stream=stream).sort_stats('cumulative')
ps.print_stats(5)
print("=== Slow tokeniser profile ===")
print(stream.getvalue())

# Benchmark both
t0 = time.perf_counter()
for _ in range(3):
    tokenise_slow(text[:10000])  # small input for slow version
slow_time = (time.perf_counter() - t0) / 3

t0 = time.perf_counter()
for _ in range(100):
    tokenise_fast(text)
fast_time = (time.perf_counter() - t0) / 100

print(f"slow: {slow_time*1000:.1f} ms (10K chars)")
print(f"fast: {fast_time*1000:.1f} ms (600K words)")
print(f"fast scales far better: O(n) vs O(n^2)")
```

```c
/* C performance profiling with perf_event (see note 1 for the full setup) */
/* Here we demonstrate measuring cache miss rate for two access patterns */
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>
#include <time.h>

#define N (1 << 22)  /* 4M int32s = 16 MB — larger than L3 */

void sequential_read(int *arr, int n) {
    long sum = 0;
    for (int i = 0; i < n; i++)
        sum += arr[i];
    (void)sum;
}

void random_read(int *arr, int *indices, int n) {
    long sum = 0;
    for (int i = 0; i < n; i++)
        sum += arr[indices[i]];
    (void)sum;
}

int main(void) {
    int *arr     = malloc(N * sizeof(int));
    int *indices = malloc(N * sizeof(int));

    /* Init with sequential indices, then shuffle for random access */
    for (int i = 0; i < N; i++) { arr[i] = i; indices[i] = i; }
    /* Fisher-Yates shuffle */
    srand(42);
    for (int i = N-1; i > 0; i--) {
        int j = rand() % (i+1);
        int t = indices[i]; indices[i] = indices[j]; indices[j] = t;
    }

    struct timespec t0, t1;

    clock_gettime(CLOCK_MONOTONIC, &t0);
    for (int rep = 0; rep < 10; rep++) sequential_read(arr, N);
    clock_gettime(CLOCK_MONOTONIC, &t1);
    long seq_ns = ((t1.tv_sec - t0.tv_sec) * 1000000000L + (t1.tv_nsec - t0.tv_nsec)) / 10;

    clock_gettime(CLOCK_MONOTONIC, &t0);
    for (int rep = 0; rep < 3; rep++) random_read(arr, indices, N);
    clock_gettime(CLOCK_MONOTONIC, &t1);
    long rnd_ns = ((t1.tv_sec - t0.tv_sec) * 1000000000L + (t1.tv_nsec - t0.tv_nsec)) / 3;

    printf("Sequential: %ld ns (%ld MB/s)\n", seq_ns,
           (long)N * 4 * 1000 / seq_ns);
    printf("Random:     %ld ns (%ld MB/s)\n", rnd_ns,
           (long)N * 4 * 1000 / rnd_ns);
    printf("Ratio:      %.1fx slower for random access\n",
           (double)rnd_ns / seq_ns);
    /*
     * Sequential: ~800 ms for 4M int32 reads → ~20 GB/s (bandwidth-bound, hits L3/DRAM)
     * Random: ~8000 ms → ~2 GB/s (latency-bound: ~200 ns × 4M = 800 ms minimum at 5ns/cache_miss)
     * This ratio (~10x) directly reflects DRAM bandwidth (~20 GB/s) vs DRAM latency (~80 ns)
     */

    free(arr); free(indices);
    return 0;
}
```

> [!TIP]
> The single most actionable use of `perf stat` output is the **IPC**. IPC ≈ 1 is average for mixed workloads on a modern OoO CPU. IPC < 0.5 almost always means memory-bound — the backend is stalling on cache misses. Check `LLC-load-misses` to confirm. IPC > 3 means compute-bound — look at whether SIMD or Tensor Cores are utilised. Use `stalled-cycles-backend` / `cycles` for the stall fraction.

Compile C: `gcc -O2 -o perf_demo perf_demo.c && perf stat -e cycles,instructions,LLC-load-misses ./perf_demo`

## In Practice

### Profiling ML Training

For PyTorch training, the profiler stack is:

```python
import torch
from torch.profiler import profile, record_function, ProfilerActivity

with profile(
    activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA],
    record_shapes=True,
    profile_memory=True,
    with_stack=True
) as prof:
    for step in range(5):
        with record_function("forward"):
            output = model(input_batch)
        with record_function("backward"):
            loss = criterion(output, target)
            loss.backward()
        with record_function("optimizer"):
            optimizer.step()
            optimizer.zero_grad()

# Print table sorted by CUDA time
print(prof.key_averages().table(sort_by="cuda_time_total", row_limit=20))

# Export to Chrome trace format (open in chrome://tracing or Perfetto)
prof.export_chrome_trace("trace.json")
```

Common ML profiling findings:
- **Data loading is the bottleneck:** `DataLoader` CPU time >> GPU compute time → increase `num_workers`, use `pin_memory=True`.
- **Attention is memory-bound for long sequences:** Switch to FlashAttention.
- **Optimizer step is slow:** Use fused optimiser (`apex.optimizers.FusedAdam`).
- **CUDA kernels are idle after each step:** Memory fragmentation or GC pauses. Use `torch.cuda.memory.max_memory_allocated()` to diagnose.

### The Performance Analysis Workflow

A disciplined analysis cycle — never skip a step:

```
1. Define metric → what does "fast enough" mean? (latency, throughput, FLOPS)
2. Measure baseline → wall time, perf stat, profiler
3. Identify bottleneck → which function, what hardware event, which resource
4. Form hypothesis → "this is cache-bound because LLC-load-misses is high"
5. Make one change → fix one thing at a time
6. Measure again → did the bottleneck metric improve? Did overall time improve?
7. Repeat
```

> [!WARNING]
> Micro-optimisations can improve the measured sub-metric but not the end-to-end performance. "I reduced the inner loop's branch mispredictions by 50%" is meaningless if the inner loop is 2% of total execution time. Always measure end-to-end latency / throughput, not just the sub-metric you targeted.

## Math

### Efficiency and Parallel Overhead

For a parallel program with N processors and parallel fraction p = 1−s:

**Parallel efficiency:** `E(N) = Speedup(N) / N`. Measures how well the N processors are utilised.

```math
E(N) = \frac{1}{N \cdot s + (1-s)} = \frac{1}{1 + s(N-1)}
```

For s=0.05 and N=100: E = 1 / (1 + 0.05 × 99) = 1/5.95 ≈ 17% efficiency. Parallelism has heavy overhead.

**Parallel overhead:** Extra time consumed by synchronisation, load imbalance, and communication.

```math
T_{overhead}(N) = N \cdot T_{parallel}(N) - T_{serial}
```

The goal is to minimise overhead while maximising parallelism — a tension that determines optimal batch sizes, chunk sizes, and communication patterns in distributed ML training.

### FLOPS vs Achieved FLOPS

Peak FLOPS × utilisation = achieved FLOPS. The utilisation gap arises from:

```math
\text{Utilisation} = \frac{\text{Achieved FLOPS}}{\text{Peak FLOPS}} = \prod_i (1 - \text{waste fraction}_i)
```

Where waste fractions include: pipeline bubbles from cache misses, branch mispredictions, register conflicts, instruction-level dependency stalls, and — for GPUs — warp divergence, memory-access pattern inefficiency, and launch overhead.

## Pitfalls

- **"Profiling overhead makes the results inaccurate."** — Statistical sampling profilers (perf, Instruments.app) have <1% overhead and are suitable for production profiling. Hardware performance counter reads (perf stat) have essentially zero overhead and give exact counts. Only instrumentation profilers (gprof, Valgrind/Callgrind) have significant overhead (10–100%) and may perturb results for timing.
- **"Optimise the slow function."** — Optimise the function with the highest product of (frequency × cost), not just the slowest. A function called 10M times that takes 100 ns is 10× more impactful than one called once that takes 1 ms.
- **"Lower CPI is always better."** — CPI < 1 is possible when superscalar execution achieves IPC > 1. But a program that runs fewer slower instructions (vector SIMD) may have higher CPI per instruction yet lower wall time than a scalar program with CPI = 0.5. Instruction count and CPI are both factors in the Iron Law.
- **"Amdahl's Law means parallelism is always limited."** — Amdahl assumes fixed problem size. For most practical workloads (larger models, larger datasets, more queries), Gustafson's Law applies: problem size scales with resources, the serial fraction shrinks, and near-linear speedup is achievable.
- **"Peak FLOPS = throughput."** — Peak FLOPS is only achievable with 100% utilisation of every functional unit, every cycle, with perfectly pipelined, fully coalesced, non-divergent code, feeding the right tile sizes to Tensor Cores. Real workloads achieve 10–60% of peak. Report achieved FLOPS.

## Exercises

### Exercise 1: Amdahl's Law applied to ML training

A distributed training job runs on 128 GPUs. The per-GPU forward+backward pass takes 800 ms. AllReduce (gradient synchronisation) takes 200 ms. Compute the speedup over a single GPU under Amdahl's model, and compute the parallel efficiency.

#### Solution

Total time per step: 1000 ms.
Serial fraction: AllReduce = 200 ms / 1000 ms = 0.20. (Actually AllReduce scales with the number of GPUs up to a point — it's not entirely serial — but treating it as serial overhead is a common approximation for this analysis.)

Actually, the AllReduce is the synchronisation overhead that doesn't benefit from more GPUs (in the simple model). So s = 0.20, N = 128.

Speedup = 1 / (0.20 + 0.80/128) = 1 / (0.20 + 0.00625) = 1 / 0.20625 ≈ **4.85×**.

Parallel efficiency = 4.85 / 128 = **3.8%**. Very poor efficiency — the synchronisation cost dominates.

Practical implication: at 128 GPUs with 20% AllReduce overhead, only 4.85× speedup is achievable over 1 GPU — not 128×. This is why gradient compression (e.g. PowerSGD, TopK sparsification), overlapping AllReduce with backward pass computation, and gradient accumulation are active research areas in distributed training.

---

### Exercise 2: Roofline analysis

A CUDA kernel processes a 1 GB array of float32 values, computing `out[i] = a * in[i] + b` (scalar multiply-add). On an H100:
- Peak FP32 CUDA core throughput: 66.9 TFLOPS
- HBM bandwidth: 3.35 TB/s

(a) How many FLOPs does the kernel perform?
(b) How many bytes are transferred?
(c) What is the arithmetic intensity?
(d) Is the kernel compute-bound or memory-bandwidth-bound?
(e) What is the achievable throughput (the tighter bound)?

#### Solution

Array size: 1 GB / 4 bytes = 256M elements.

**(a) FLOPs:** Each element performs 2 FLOPs (1 multiply + 1 add). Total: 256M × 2 = **512M FLOPs = 0.512 GFLOPs**.

**(b) Bytes transferred:** Read 1 GB (input), write 1 GB (output). Total: **2 GB**.

**(c) Arithmetic intensity:** AI = 0.512 × 10⁹ FLOPs / 2 × 10⁹ bytes = **0.256 FLOPs/byte**.

**(d) H100 ridge point:** 66.9 TFLOPS / 3.35 TB/s ≈ 20 FLOPS/byte.
AI = 0.256 << 20 → **memory-bandwidth-bound** (by a factor of ~80×).

**(e) Achievable throughput:**
Memory bound: min bound = bandwidth × AI = 3.35 TB/s × 0.256 = **858 GFLOPS** (if memory bandwidth is fully utilised).
Compute bound: 66.9 TFLOPS. Memory bound is tighter.
Optimal throughput: **858 GFLOPS** (if achieving 100% of 3.35 TB/s bandwidth).

This is a classic "streaming kernel" — almost all ML element-wise operations (GELU, LayerNorm, residual add) are this type. At 2 FLOPs per element, the kernel needs to sustain 2/3.35 × 10³ ≈ 0.6 ns per byte — essentially just the HBM bandwidth. Kernel fusion (combining multiple such operations into one pass) reduces memory traffic proportionally.

---

### Exercise 3: Profiling interpretation

`perf stat` output for a C++ program:

```
       3,142,512,048      cycles
       1,247,894,325      instructions   # 0.40 insn per cycle
          98,123,441      LLC-load-misses
           4,234,521      branch-misses
     1,234,987,123      stalled-cycles-backend # 39.3% of cycles
```

Diagnose the bottleneck and suggest one specific fix.

#### Solution

**Diagnosis:**

- **IPC = 0.40** (instructions/cycles = 1.25B/3.14B): very low. A well-running program on a modern OoO CPU expects IPC ≥ 1.
- **stalled-cycles-backend = 39.3%**: Nearly 40% of cycles the backend execution units are idle — strongly suggests memory-bound execution.
- **LLC-load-misses = 98M**: Very high. If each LLC miss costs ~300 cycles at DDR5 latency, total miss cycles ≈ 98M × 300 = 29.4B cycles — exceeding even total cycles (3.14B), which means these are overlapping (out-of-order memory-level parallelism). Still, 98M LLC misses indicate frequent DRAM access.
- **branch-misses = 4.2M**: Moderate. At 15 cycles/miss, penalty = 63M cycles ≈ 2% of total cycles. Not the primary issue.

**Conclusion:** The program is heavily memory-bound. 39% of cycles are backend stalls, dominated by LLC cache misses going to DRAM.

**Suggested fix:** Profile which data structure causes the LLC misses (`perf mem record` or `perf c2c`). Most likely: a data structure with poor spatial locality (linked list, tree traversal) or a working set larger than the LLC. Fix options:
1. **Restructure data layout** to improve spatial locality (array-of-structs → struct-of-arrays, or smaller struct sizes).
2. **Reduce working set** to fit in L3 (e.g. work on smaller sub-problems at a time).
3. **Software prefetching** (`__builtin_prefetch()`) for predictable access patterns.

---

### Exercise 4: Gustafson vs Amdahl

A molecular dynamics simulation has a 2% serial setup fraction (initialisation and I/O). On 16 CPUs, it runs in 100 seconds. On 256 CPUs, it runs in 20 seconds.

(a) Is this consistent with Amdahl's Law or Gustafson's Law? Explain.
(b) Compute the theoretical Amdahl speedup for N=256, s=0.02.
(c) Compute the Gustafson scaled speedup for N=256, s=0.02.
(d) Why might the actual measured speedup differ from both predictions?

#### Solution

**Observed speedup (16→256 CPUs):** we don't have the single-CPU time, but the ratio 256/16 CPUs to 100s/20s suggests approximate linear scaling.

Let's interpret: if on 16 CPUs = 100s, and assuming roughly ideal parallel scaling from 1 CPU: T(1) = 16 × 100s ≈ 1600s (rough). T(256) = 20s. Observed speedup(1→256) ≈ 1600/20 = **80×**.

**(a) Consistency with laws:**
Amdahl predicts max 1/0.02 = 50× speedup regardless of N. The observed 80× exceeds this. Amdahl's prediction is violated.
Gustafson's model (weak scaling — problem grows with resources) allows the speedup to scale as N − s(N−1). With N=256, s=0.02: 256 − 0.02 × 255 = 256 − 5.1 = **250.9×** maximum scaled speedup. The observed 80× is consistent with Gustafson's framework (80× < 250×).

**Conclusion:** The simulation follows Gustafson's weak scaling — as more CPUs are added, the simulation domain (number of molecules) is scaled up proportionally, and the 2% serial fraction shrinks relative to total work.

**(b) Amdahl speedup at N=256, s=0.02:**
Speedup = 1 / (0.02 + 0.98/256) = 1 / (0.02 + 0.00383) = 1 / 0.02383 ≈ **42×**

**(c) Gustafson scaled speedup at N=256, s=0.02:**
Scaled speedup = 256 − 0.02 × (256−1) = 256 − 5.1 = **250.9×**

**(d) Why actual speedup might differ:**
- Load imbalance: not all CPUs have equal work (e.g. boundary regions in domain decomposition).
- Communication overhead: inter-CPU message passing (MPI) is not captured in s.
- Hardware effects: NUMA latency, network topology bottlenecks, bandwidth contention.
- The serial fraction s itself may vary with N (not a constant 2% at all scales).

---

### Exercise 5: Performance counter-driven optimisation

You observe that a training loop achieves 200 GFLOPS on an H100 that is capable of 2000 TFLOPS in BF16 Tensor Core mode — a **0.01 utilisation** (1%). Using the roofline framework, determine whether this is compute-bound or memory-bound, and describe three specific hardware-architectural reasons why Tensor Cores might be underutilised.

#### Solution

**Roofline classification:**
The program achieves 200 GFLOPS = 0.2 TFLOPS. The H100 ridge point (BF16 Tensor Core) ≈ 597 FLOPS/byte. If the achieved FLOPS are 200 GFLOPS and we don't know bandwidth utilisation, we need the arithmetic intensity.

If the kernel is truly compute-bound at the architecture's ridge (600 FLOPS/byte), the expected bandwidth would be 200 GFLOPS / 600 FLOPS/byte = 0.33 GB/s — far below H100's 3.35 TB/s peak. This implies the kernel is neither achieving its compute ceiling nor its bandwidth ceiling.

**Three reasons for Tensor Core underutilisation:**

1. **Non-multiple-of-16 matrix dimensions:** Tensor Cores operate on 16×16×16 tiles. If the model has attention head dimensions or batch sizes that are not multiples of 16, the tiles are zero-padded internally or fall back to CUDA cores. Dimensions like 13, 17, or 3 cause near-zero Tensor Core utilisation. **Fix:** Pad dimensions to multiples of 64 or 128.

2. **Excessive non-matmul operations in the kernel:** If the kernel alternates matmul with memory-bound operations (e.g. per-element softmax, LayerNorm, residual adds between small matmuls), the Tensor Cores are idle during the non-matmul phases. At 200 GFLOPS vs 2000 TFLOPS peak, only 10% of cycles would need to be in matmul with 100% Tensor Core utilisation. **Fix:** Fuse operations (as FlashAttention does) so the non-matmul work is hidden within the matmul computation.

3. **Insufficient warp occupancy / pipeline stalls:** Tensor Cores have long latency (4–8 cycles). With low occupancy (few active warps), the warp scheduler cannot hide this latency with other warps' work, leaving the Tensor Cores idle between issues. **Fix:** Increase thread block size or reduce register pressure to increase SM occupancy; profile with `ncu --set full` to see `sm__pipe_tensor_cycles_active.avg.pct_of_peak_sustained_active`.

## Sources

- Amdahl, G. M. (1967). "Validity of the single processor approach to achieving large scale computing capabilities." *Proceedings of the AFIPS Spring Joint Computer Conference*. https://dl.acm.org/doi/10.1145/1465482.1465560
- Gustafson, J. L. (1988). "Reevaluating Amdahl's Law." *Communications of the ACM*, 31(5), 532–533. https://dl.acm.org/doi/10.1145/42411.42415
- Williams, S., Waterman, A., & Patterson, D. (2009). "Roofline: An Insightful Visual Performance Model for Multicore Architectures." *Communications of the ACM*, 52(4), 65–76. https://dl.acm.org/doi/10.1145/1498765.1498785
- Gregg, B. *Systems Performance: Enterprise and the Cloud* (2nd ed., 2020). Addison-Wesley Professional.
- Intel VTune Profiler Documentation. https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/current/overview.html
- Brendan Gregg's perf examples. https://www.brendangregg.com/perf.html

## Related

- [1 - What is Computer Architecture](./1-what-is-computer-architecture.md)
- [5 - Pipelining and Hazards](./5-pipelining-and-hazards.md)
- [6 - Memory Hierarchy and Caches](./6-memory-hierarchy-and-caches.md)
- [10 - GPUs and Accelerators](./10-gpus-and-accelerators.md)
