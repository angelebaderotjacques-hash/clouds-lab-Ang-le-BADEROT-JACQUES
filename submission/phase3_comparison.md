# Phase 3 — Comparison Writeup

## 1. Facts from your runs

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->66.7<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.007<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->3.2<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->7<!--/answer--> |

**Explain the timings.**

<!--answer:timing_explanation-->
Local single-process word count (0.007 s) is fastest because everything runs in memory in one process with zero network or orchestration overhead. Spark TF-IDF (3.2 s) is second despite computing more work (TF, DF, join, and Parquet/collect) because it starts a local JVM/Spark runtime on one machine. Cloud MapReduce (66.7 s) is slowest due to Cloud Workflows orchestration, Cloud Run invocations, GCS I/O, and chaos retries. Comparing local vs. cloud word count is like-for-like; most cloud time goes to distributed coordination overhead rather than CPU work.
<!--/answer-->

**What the counts tell you.**

<!--answer:counts_interpretation-->
Four stages correspond to three shuffle boundaries in `spark_tfidf.py`: `reduceByKey` in `term_freq`, `reduceByKey` in `doc_freq`, and `join` in `build_tfidf_rdd` (plus `collect`). Seven retries out of ~16 task attempts (~44%) is slightly above the expected 30% rate from `--chaos 0.3` on a small sample.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.**

<!--answer:transformations_vs_actions-->
In `spark_tfidf.py` and the notebook, `parallelize`, `flatMap`, `map`, `reduceByKey`, and `join` are lazy transformations: calling `build_tfidf_rdd` only records the RDD lineage graph without computing results. `collect` is the action that materializes the DAG into stages and tasks, executes them, and returns the 5,975 TF-IDF rows to the driver.
<!--/answer-->

**Lineage and recovery.**

<!--answer:lineage_recovery-->
If a worker holding a partition from the first `reduceByKey` (`ShuffledRDD[3]`, stage 0) died, Spark would recompute only the lost partition(s) along the upstream lineage from `ParallelCollectionRDD[0]` and `map_wc`/tokenization, reusing shuffle outputs already written and keeping unaffected partitions/stages intact, starting recovery at that shuffle boundary.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows (`mr-wordcount`, defined in `phase-3-mapreduce-spark/workflow.yaml`)<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->Cloud Run function (`mr-worker`, implemented in `phase-3-mapreduce-spark/main.py` using `mapreduce.py`)<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Google Cloud Storage (`<project>-eurecomgpt` bucket with `<job>/input/`, `shuffle/`, and `output/` prefixes)<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->`shuffle` / partition assignment logic in `phase-3-mapreduce-spark/mapreduce.py` (routing keys across `num_reducers` partitions)<!--/answer-->

**The barrier.**

<!--answer:barrier_location-->
The barrier is the `map_phase` step using `parallel` with `for` in `workflow.yaml`: Cloud Workflows waits for every parallel map branch to finish before proceeding to `reduce_phase`.
<!--/answer-->

**Idempotence.**

<!--answer:idempotence-->
Overwriting shuffle objects at deterministic keys makes retries safe because re-running a failed map task produces the exact same state. If `run_map` appended instead of overwriting, retried map attempts would duplicate `(key, value)` pairs in shuffle files, inflating word counts and breaking exactness.
<!--/answer-->

**The chaos run.**

<!--answer:chaos_observations-->
With `--chaos 0.3`, the execution log and Logs Explorer showed 7 HTTP 503 simulated worker crashes retried automatically with exponential backoff by Cloud Workflows. The job still succeeded (66.7 s) and returned the exact same 2,221 distinct words and top terms as the local reference.
<!--/answer-->
