# Phase 3 :Comparison Writeup

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
The local run (18 ms) is the fastest because everything runs in one process with no network. The cloud MapReduce (23.7 s) does the same word count but sends tasks to 12 workers on Cloud Run and uses GCS to pass data between map and reduce, so most of the time is waiting for network and HTTP calls, not computing. Spark (7.0 s) does more work (TF-IDF has two aggregations and a join) but runs on one machine, so it is faster than the cloud job. The local vs cloud comparison is fair (both do word count); Spark is a different job so the numbers cannot be compared directly.
<!--/answer-->

**What the counts tell you.**

<!--answer:counts_interpretation-->
4 stages means 3 shuffles happened. The first shuffle was after term_freq (reduceByKey in spark_tfidf.py:32), the second after doc_freq (line 45), and the third during the join (line 58). With --chaos 0.3, about 30% of the 16 tasks were expected to fail randomly, so around 5 retries is normal and matches what we saw.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.**

<!--answer:transformations_vs_actions-->
flatMap, map, reduceByKey, and join are all transformations. They just build a plan and nothing runs yet. collect is the only action and it triggers all the computation. When we called build_tfidf_rdd, Spark only recorded the steps. When we called collect, Spark actually ran all 4 stages and returned the results.
<!--/answer-->

**Lineage and recovery.**

<!--answer:lineage_recovery-->
If a partition from the first reduceByKey (Stage 0) was lost, Spark would only recompute that partition by going back to the original data and re-running flatMap and reduceByKey for it. The other partitions are already saved and do not need to be recomputed. Recovery starts from Stage 0.
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
In workflow.yaml, all map tasks run inside a `parallel` block. Cloud Workflows waits for all branches to finish before moving to the next step. This is the barrier, at the end of the `parallel` block.
<!--/answer-->

**Idempotence.**

<!--answer:idempotence-->
Each map task writes its output to a GCS file named with its task ID, and it overwrites the file each time. So if a task is retried, it produces the same file again with no duplicates. If it appended instead of overwriting, a retry would write the data twice and the word counts would be wrong.
<!--/answer-->

**The chaos run.**

<!--answer:chaos_observations-->
With --chaos 0.3, some tasks failed randomly. Cloud Workflows retried them automatically. The job still finished correctly with 2221 words, the same as the local result. This shows the retry logic works and the job can handle failures.
<!--/answer-->
