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

-**

### References:
1. https://copyconstruct.medium.com/monitoring-and-observability-8417d1952e1c
2. https://softwareengineeringdaily.com/2021/02/04/debunking-the-three-pillars-of-observability-myth/
3. https://thenewstack.io/observability-wont-replace-monitoring-because-it-shouldnt/
4. https://charity.wtf/category/observability/







