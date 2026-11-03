# Phase 2 — Reflection: where does parallelism stop helping?

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->474.401<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.012<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->464656.3<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->12964.3<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4541<!--/answer--> |

**The thread sweep.**

<!--answer:thread_sweep-->
1 thread: 4453.188 ms
2 threads: 2252.165 ms
4 threads: 2410.895 ms
8 threads: 2316.859 ms
Fastest: 2 threads
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.**

<!--answer:vectorization_why-->
Python loops are slow because the interpreter handles each operation one by one. NumPy
sends the whole matrix to OpenBLAS, which uses SIMD instructions to process many numbers
at once. Same result, but 468x faster because the CPU is actually used efficiently.
<!--/answer-->

**Where the thread curve flattens, and why.**

<!--answer:threading_plateau-->
It stopped helping after 2 threads. The Colab machine only has 2 real CPU cores, so extra
threads just share the same cores and waste time coordinating. Going from 2 to 4 threads
actually got slightly slower (2252ms to 2411ms).
<!--/answer-->

**What made the GPU faster.**

<!--answer:gpu_why-->
Transformer training is mostly matrix multiplications. The T4 has thousands of cores that
do these in parallel, giving 35.8x speedup over CPU. A workload with very small batches
would not benefit because the data transfer cost to the GPU would cancel out the gain.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?**

<!--answer:parallelism_limit-->
Each experiment hit a different wall. Vectorization was limited by Python interpreter
overhead — no amount of extra cores fixes that, only leaving Python does (474ms to 1ms).
Threading was limited by hardware: 2 real cores means 2 threads is the ceiling, and going
higher just adds coordination cost (2252ms at 2 threads vs 2411ms at 4). The GPU was fast
(35.8x) because the matrix operations were large enough to keep it busy, but small batches
would shrink that advantage since the CPU-to-GPU transfer cost stays fixed.
<!--/answer-->

**What you would measure next.**

<!--answer:next_experiment-->
Sweep batch size on the GPU from 1 to 256 and measure speedup over CPU at each size. This
would show exactly where the GPU stops being worth it, which would sharpen the conclusion
about when parallelism helps.
<!--/answer-->
