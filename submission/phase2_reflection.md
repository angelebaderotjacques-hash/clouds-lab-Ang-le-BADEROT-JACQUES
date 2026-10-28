# Phase 2 — Reflection: where does parallelism stop helping?

Copy this file to `submission/phase2_reflection.md`, fill in every answer slot, and commit it:

```bash
cp phase-2-tiny-gpt/reflection_template.md submission/phase2_reflection.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or your threading plot freely.

Numbers in section 1 must **match `submission/phase2_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->229.946<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.025!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->400725.8<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->12513.8<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4759<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2184.043 ms, 2 threads: 2030.216 ms, 4 threads: 2311.072 ms, 8 threads: 2044.146 ms. The fastest count was 2 threads.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive version spends its time executing explicit Python loops, causing constant interpreter overhead, dynamic type checking, and poor CPU cache locality. The vectorized version (`229.95 ms` down to `1.03 ms`) hands contiguous memory blocks directly to BLAS libraries. These leverage CPU hardware AVX/SIMD vector extensions, processing multiple floating-point operations in parallel within single CPU cycles using vectorized registers.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
Performance stopped improving beyond 2 threads, with 4 threads even degrading performance to `2311.07 ms`. Colab's standard runtime provides a limited number of physical CPU cores (2 vCPUs). Beyond the physical core count, additional threads introduce heavy synchronization overhead, context-switching penalties, and cache contention on shared L3 caches. Managing thread synchronization costs more than the marginal parallelism gained.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
Transformers are dominated by dense matrix multiplications and batched GEMM operations involving independent parameters, perfectly matching the thousands of Arithmetic Logic Units (ALUs) inside the T4 GPU (`12514.8 ms` for 100 steps vs `400725.8 ms` on CPU). Conversely, heavily sequential algorithms with tight loop-carried dependencies (such as standard iterative parsing or text tokenization loops) would see little to no speed-up.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Parallelism hits different walls depending on the hardware layer. For SIMD vectorization, hardware width limits acceleration, yet it achieved a massive speedup (`1.025 ms` vs `229.946 ms`), bound only by unvectorizable sequential instructions. For multi-threading, CPU core limits and coordination costs bound performance: pushing past 2 threads increased latency (`2311.072 ms` at 4 threads) because thread contention outweighed parallel gains on a 2-vCPU machine. Finally, GPU training achieved a `32.0x` speedup (`12514.8 ms` GPU vs `400725.8 ms` CPU), but is ultimately bound by host-device memory bandwidth and kernel launch overheads when batch sizes or model dimensions are too small to keep all CUDA cores saturated.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
Measuring training step time across varying micro-batch sizes on the GPU. Finding that step time remains constant despite reducing batch size would change my mind, proving that memory bandwidth or kernel launch overheads dominate over raw ALU compute capacity.
<!--/answer-->
