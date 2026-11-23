# Phase 3 — Comparison Writeup

## 1. Facts from your runs

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->23.7<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.018<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->7.0<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->5<!--/answer--> |

**Explain the timings.**

<!--answer:timing_explanation-->
The local single-process run (18 ms) is fastest because all three MapReduce primitives run in one Python process with no network overhead. The cloud MapReduce (23.7 s) performs the same word count but fans out across 12 map workers and 4 reducers on Cloud Run, paying the cost of HTTP invocations, GCS shuffle files, and Cloud Workflows orchestration. The Spark TF-IDF (7.0 s) is not like-for-like — it computes a heavier job (two reduceByKeys and a join) on a single local machine. That Spark finishes TF-IDF in 7 s while the cloud job spends 23.7 s on simpler word count shows that most of the cloud job time is network and orchestration overhead, not computation.
<!--/answer-->

**What the counts tell you.**

<!--answer:counts_interpretation-->
Four stages mean three shuffle boundaries. Stage 0 ended at the term_freq reduceByKey (spark_tfidf.py:32), Stage 1 at the doc_freq reduceByKey (line 45), and Stage 2 at the join (line 58); Stage 3 ran collect. With --chaos 0.3 and 16 total task slots, roughly 30% — about 5 tasks — were expected to fail and be retried, which matches what the workflow execution log showed.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.**

<!--answer:transformations_vs_actions-->
In spark_tfidf.py, flatMap, map, reduceByKey (called twice), and join are all transformations — they build a lazy DAG without executing anything. The collect in the notebook is the only action. When build_tfidf_rdd ran, Spark simply recorded the computation graph and nothing executed. When collect was called, Spark triggered all four stages in sequence, actually computing term frequencies, document frequencies, the join, and the final TF-IDF scores.
<!--/answer-->

**Lineage and recovery.**

<!--answer:lineage_recovery-->
If the worker holding an output partition of the first reduceByKey (Stage 0, line 32) died, Spark would recompute only that partition: re-run flatMap and reduceByKey from the original parallelized RDD. All other Stage 0 partitions already written to shuffle storage could be reused. Recovery restarts from Stage 0's shuffle boundary, leaving Stages 1–3 unaffected until their inputs are needed.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows (`mr-wordcount`)<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->Cloud Run / Cloud Functions gen2 (`mr-worker`)<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Cloud Storage (GCS bucket `eurecomgpt-lab-511412-eurecomgpt`)<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->`zlib.crc32` hash in the shuffle step of `mapreduce.py`<!--/answer-->

**The barrier.**

<!--answer:barrier_location-->
The map tasks all run inside a `parallel` block in workflow.yaml. Cloud Workflows implicitly waits for every branch of a `parallel` block to complete before advancing to the next step, so the barrier is the closing of that `parallel` block.
<!--/answer-->

**Idempotence.**

<!--answer:idempotence-->
Each map task writes its shuffle pairs to a GCS file named after its task ID, overwriting any previous content. A retry therefore produces exactly the same file, so no duplicates arise. If `run_map` appended instead of overwriting, a retried task would add a second copy of its pairs to the shuffle file, causing each key's count to be doubled and corrupting the final word counts.
<!--/answer-->

**The chaos run.**

<!--answer:chaos_observations-->
With --chaos 0.3, roughly 30% of tasks raised a random exception. Cloud Workflows caught each failure and retried the affected step with exponential backoff (up to 5 attempts). The job still completed successfully with 2221 distinct words, identical to the local reference, confirming that the retry policy is sufficient to tolerate the injected fault rate.
<!--/answer-->
