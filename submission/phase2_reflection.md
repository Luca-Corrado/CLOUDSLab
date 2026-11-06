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
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->355.54<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->11.985<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->464054.3<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13121.3<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.5001<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
2250.701, 2022.36, 2523.956 , 2447.428
The fastest count was 2 threads.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive attention version relies on Python loops to navigate the sequence, computing one dot product at a time. This sequentially bounds the process and causes high interpreter overhead. 
In contrast, the vectorized version offloads the operations to NumPy, which uses a BLAS library compiled with AVX/SIMD instructions to process multiple floating-point numbers with a single vector instruction.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
The threading sweep reaches peak performance at 2 threads, before degrading at 4 and 8 threads. Pytorch tries to split a single CPU operation across these worker threads. However, the Colab CPU instance only provides 2 vCPUs. 
Beyond this count, the physical hardware to execute the threads runs out, forcing them to compete for the same two cores. This causes actions such as the overhead of context switching and memory coordination to cost more time.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
A GPU accelerates this workload because it possesses thousands of execution units. A transformer training step is composed of many matrix multiplications, that maps perfectly to parallel execution. 
A workload that would not see this speed-up is one reliant on sequential branch logic or recursive algorithms, such as irregular tree structures, where each computational step waits on the outcome of the immediatly preceding step. 
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Parallelism has different limits depending on the workload. In the threading experiment, performance peaked at 2 threads (2022.36 ms), while 4 threads increased execution time to 2523.956 ms. This suggests that limited hardware resources and coordination overhead can outweigh the benefits of additional threads. 
In the vectorization experiment, execution time dropped from 355.54 ms to 11.985 ms. However, not all work can be vectorized, since sequential dependencies and the overhead of dispatching operations still limit performance. 
Finally, GPU training took 13121.3 ms for 100 steps, compared with 464054.3 ms on CPU. This demonstrates how GPUs exploit the parallelism of matrix operations, although hardware resources and data movement can eventually limit further scaling. 
These experiments show that parallelism stops helping when insufficient hardware, irreducible sequential work, coordination costs or data movement become the dominant constraints.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would measure GPU training time with and without host-to-device data transfers, keeping the model and workload identical. If keeping the data on the GPU significantly reduced the total training time compared with transferring batches at every step, it would show that data movement is an important bottleneck. Otherwise, computation or other overheads would likely dominate.
<!--/answer-->
