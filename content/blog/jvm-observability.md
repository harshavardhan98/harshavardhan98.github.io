+++
title = "I know what my JVM did last Release?"
date = "2025-01-15"
description = "A deep dive into the JVM metrics that actually matter after a release — and how to read them like a detective."
tags = [ "observability", "JVM"]
+++

It's 3 PM on a Friday and you plan to deploy the weekend change. You merged the PR, the pipeline is green and deployment is successful.
Everything looks fine until you receive a Slack message regarding "API latency is a bit high and the customers are complaining?"

You open your Grafana dashboard and you notice that the P99 lateny is slightly higher than usual but you are not sure what is the root cause. This post is a walk through of the JVM metrics you need to check and how to interpret them to find the root cause of the issue. I will try to provide an intuitive understanding of what "normal" graph shape looks like and what shapes on graph should make you nervous. 
The metrics discussed in this post are the standard metrics exposed by the JVM prometheus exporter/OTEL agent.

## 1. Memory & Garbage Collection
This is where we need to start checking first. If the release introduced a memory leak, changed object allocation pattern, or accidentally upgraded a library which holds reference longer, it will show up here first

### Heap Memory:
The JVM splits the heap into generations. The metric to look out for:

- **jvm_memory_used_bytes{area="heap"}** - This metric tells the total heap in use. Compare the baseline before and after the release. A slow upward trend that doesn't comeback down after GC is a classic memory leak shape.
- **jvm_memory_used_bytes{area="Eden Space"}** - Eden space is the region where new objects are born. High churn here is normal. What we are looking for is where eden is filling up *faster* than before the release
- **jvm_memory_used_bytes{area="Old Gen"}** - Old Gen contains objects that survived multiple GC cycles. If old gen is climbing steadily and not being reclaimed then this points to a leak or retention issue. This is the metric that pages you at 3 AM.
- **jvm_memory_used_bytes{area="Survivor Space"}** - This is the waiting room between eden and old gen. If survivor space is consistently full, objects are being promoted to old gen too aggressively.

### Non Heap Memory:
Many In-Memory databases like Apache ignite uses off-heap memory for caching and storage. Libraries like Spring, ByteBuddy etc., which uses reflection and new class generation during run-time will lead to metaspace growth. Libraries like Netty and NIO libraries allocate direct buffer for performance which if not released properly can result in consuming all available native memory. High off-heap memory usage will catch us off-guard and it will affect other processes running in the host.

**jvm_memory_used_bytes{area="non_heap"}** - This metrics tells the space occupied by Metaspace, code cache, compressed class space. If this is growing unbounded, it indicates a clasloader leak common for hot-reload frameworks or libraries that heavily use reflection.

**jvm_memory_used_bytes{area="Metaspace"}** - Stores class metadata. A release that pulls in a fat new dependency or uses lot of dynamic proxies like Spring can cause metaspace to ballon. Metaspace doesn't have a fixed cap by default, so it can eat into system memory quietly.

**Garbage Collection Latency Measurement**:
GC metrics tells how hard JVM is working to tame the memory

-**Average GC duration**: jvm_gc_pause_seconds_sum / jvm_gc_pause_seconds_count - total time spent in GC divided by the number of GC collections provides the average GC pause time. A sustained increase in average GC pause is a red-flag.

-**Max GC Pause**: This metric will provide the highest GC pause in the window. If this spikes from 50ms to 500ms, this will have a huge impact on the overall latency of the API

### PromQL Queries:
```PromQL
# Old gen usage over time - look for upward-trending sawtooth
jvm_memory_used_bytes{id="Tenured Gen"} / jvm_memory_max_bytes{id="Tenured Gen"}

# Average GC pause duration (5m window)
rate(jvm_gc_pause_seconds_sum[5m]) / rate(jvm_gc_pause_seconds_count[5m])

# GC Frequency - number of collections per minute
rate(jvm_gc_pause_seconds_count[5m]) * 100

# Heap Usage Ratio - set altert if above 85%
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}
```

### Good references for understanding JVM, GC and memory monitoring:
[Plumbr's blog on memory leak](https://medium.com/@plumbr/memory-leaks-fallacies-and-misconce-8c79594a3986)
[Garbage Collection Patterns](https://blog.gceasy.io/interesting-garbage-collection-patterns/)
[Uber JVM's memory tuning](https://www.uber.com/en-IN/blog/jvm-tuning-garbage-collection/)
[Datadog GC deep dive](https://www.datadoghq.com/blog/understanding-java-gc/)

## 2. Thread Metrics
Thread metrics are the vital signs of your application's concurrency model. A release that introduces a blocking calls, lock contention issue or thread pool misconfiguration can be diagnozed using the below metrics.

**jvm_threads_live_threads** - This metric gives the total active threads. If this jumps post-release and doesn't come back down, you are probably creating a lot of threads without proper shutdown

**jvm_threads_daemon_threads** - Daemon threads die when the JVM exits. A sudden increase usually means a new background task or connection pool was introduced.

**jvm_threads_peak_threads** - This metric provides the peak thread count since the JVM start. Helps us understand application's burst behaviour and capacity planning. If the value of the metrics is consistently close to a hard limit, it may indicate a need to adjust thread pool configurations.

**jvm_threads_states_threads** - This metrics provides the number of threads which are in different states - RUNNABLE, BLOCKED, WAITING/TIMED_WAITING

RUNNABLE - number of threads which are actively doing work
BLOCKED  - number of threads waiting to enter a synchronized block. If this is high, you have lock contention. A release that adds a   synchronized keyword in ahot path can tank throughput
WAITING/TIMED_WAITING - number of threads waiting on I/O, sleep, or condition variables. Normal for thread pools sitting idle. Abnormal if the count is climbing - means threads are waiting and never coming back

### PromQL Queries:
```PromQL
# BLOCKED thread count - should be near zero
jvm_threads_states_threads{state="blocked"}

# Thread count growth rate - should be flat in steady state
deriv(jvm_threads_live_count[30m])

# Ratio of BLOCKED to total threads - alert if > 0.1
jvm_threads_states_threads{state="blocked"} / jvm_threads_live_threads

# Waiting threads - compared to baseline
jvm_threads_states_threads{state="waiting"} + jvm_threads_states_threads{state="timed-waiting"}
```

**Patterns to fear**: BLOCKED thread goes from near zero to double digits. Throughput will be affected significantly even though CPU usage might be low when there is high thread contention. 


### References for understanding threads and its states
[Java Thread states](https://blog.fastthread.io/java-suspended-thread-states-blocked-waiting-timed_waiting/)
[Java Thread Dump Analysis](https://dzone.com/articles/how-analyze-java-thread-dumps)

### References:
1. https://copyconstruct.medium.com/monitoring-and-observability-8417d1952e1c
2. https://softwareengineeringdaily.com/2021/02/04/debunking-the-three-pillars-of-observability-myth/
3. https://thenewstack.io/observability-wont-replace-monitoring-because-it-shouldnt/
4. https://charity.wtf/category/observability/







