# Phase 3 — Comparison Writeup

Copy this file to `submission/phase3_comparison.md`, fill in every answer slot, and commit it:

```bash
cp phase-3-mapreduce-spark/comparison_template.md submission/phase3_comparison.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers in section 1 must **match `submission/phase3_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Facts from your runs

Numbers only (e.g. `12`, `20.3`, `0.041`).

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->44.7<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.008<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->4.2<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->5<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
Ranked from fastest to slowest: local single-process word count (0.008 s), Spark TF-IDF (4.2 s) and cloud MapReduce word count (44.7 s). The like-for-like comparison is local vs. cloud MapReduce because both perform the same word-count task on 60 documents. The local version runs in memory, while the cloud version incurs distributed execution overhead from Workflows, HTTP calls to Cloud Run, worker startup and Cloud Storage shuffle I/O. Spark performs a heavier TF-IDF computation, including two reduceByKey operations and a join, yet finishes much faster than cloud MapReduce. This suggests that orchestration and network/storage overhead contribute substantially to the cloud job's wall time.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
The four Spark stages correspond to three shuffle boundaries in the reported lineage: the reduceByKey in term_freq, the reduceByKey in doc_freq and the join in build_tfidf_rdd. The cloud job had 16 original tasks (12 map and 4 reduce) and 5 retry attempts, for 21 attempts in total. Retries were 5/21, or about 24% of all attempts; compared with the 16 original tasks, they represent about 31% additional attempts. This is reasonably consistent with a 0.3 injected-failure probability, allowing for randomness.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
The pipeline uses flatMap, reduceByKey, map and join as transformations. Parallelize creates the initial RDD. These operations build the RDD lineage without immediately executing the computation. Calling build_tfidf_rdd returns the resulting RDD. Calling collect() triggers the job execution, runs the required stages and tasks, and brings all 5975 TF-IDF rows to the driver as a Python list, meaning that collect() is the action.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
The first reduceByKey is Stage 0 and produces the shuffle output represented by ShuffledRDD[3]. If a worker loses one of its output partitions, Spark can recompute the missing partition by following the RDD lineage back to ParallelCollectionRDD[0] and rerunning the required Stage 0 computation. It can reuse surviving shuffle partitions that are still available. Recovery thus starts at the first shuffle boundary, rather than restarting the entire job.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows (phase-3-mapreduce-spark/workflow.yaml, workflow mr-wordcount)<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->Cloud Run functions (phase-3-mapreduce-spark/main.py, service mr-worker)<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Google Cloud Storage (bucket gs://cloudslab-511508-eurecomgpt)<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->shuffle() / hash(word) % num_reducers in phase-3-mapreduce-spark/mapreduce.py<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The barrier is provided by the parallel step containing the for loop in map_phase in workflow.yaml. Cloud Workflows waits for all parallel map branches, including their retries, to finish before proceeding to the next step, reduce_phase.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
Each run_map call deterministically reads a fixed split (split_{i}.json) and overwrites deterministic GCS paths (shuffle/map_{i}_part_{r}.json), so retrying a failed task produces the exact same shuffle files. If run_map appended instead of overwriting, a retried task after a transient error would duplicate its (word, count) pairs in the shuffle file, wrongly increasing the final word counts.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
With --chaos 0.3, the Cloud Run worker logs showed 5 simulated HTTP 503 failures among the 21 requests shown, alongside 16 HTTP 200 responses. Cloud Workflows' retry policy allowed the job to recover from the transient failures. The chaos run ultimately finished with state SUCCEEDED and the final word-count output matched the local reference.
<!--/answer-->
