+++
title = "What is Observability?"
date = "2024-09-18"
description = "Gentle Introduction to observability"
tags = [ "observability"]
+++

Observability is a term borrowed from the field of control systems where we measure the internal state of the system based on the external output. In the context of software, observability refers to the ability to understand what is happening inside an application or infrastructure by collecting, correlating, and analyzing key telemetry data - metrics, logs and traces.

Observability is not the same as monitoring. Monitoring is more of a symptom based mechanism where we collect predefined metrics and check if system is running as expected. Observability, on the other hand, is broader and more dynamic. It allows you to explore unknowns, debug, and understand the system behavior, especially in scenarios that weren’t anticipated when the system was built. Monitoring tells us if a system was wrong but Observability helps you find out why it's wrong and helps you reason about your system performance.


### Telemetry Data for Observability

To build a fully observable system, we rely on three key types of data:

**Logs:** Logs are detailed records of discrete events that occur within a system. They provide context and insights into individual actions or errors, making them crucial for debugging. Logs tell you what happened at a particular point in time.
Example: "2024-10-05 12:34:56 ERROR Payment service failed due to timeout"

**Metrics:** Metrics are numerical measurements that track the performance and health of systems over time. Metrics tell you how well the system is performing and are often used for system health monitoring, setting alerts, and tracking trends.
Example: CPU utilization, memory usage, request latency.

**Traces:** Traces track the journey of a request or transaction as it travels through various services within a system. They provide insights into how different components of a system interact and help you pinpoint where delays or failures might be occurring.
Example: A trace might show that a request took 500ms, with 400ms spent in the database.

Together, logs, metrics, and traces form the core data types needed for achieving observability. These pillars allow you to ask questions about the behavior of your system, whether they are about performance, errors, or operational bottlenecks.

### Why Does Observability Matter?

As systems become more distributed and complex, the number of potential failure points increases. In such environments, simple monitoring tools may not be enough. Observability gives engineers the tools to:

**Understand System Health:** Observability helps you get a clear picture of how your system is functioning in real-time. It allows you to spot performance bottlenecks, errors, or inefficiencies.

**Improve Troubleshooting:** When something goes wrong, observability helps you dig deeper into the issue, understand its root cause, and fix it faster. This leads to reduced downtime and better reliability.

**Predict Failures:** By analyzing historical data, observability helps teams identify patterns and predict potential failures before they happen, leading to proactive maintenance.

**Empower Teams:** With the right observability tools in place, teams can independently debug issues, reducing dependencies on specific experts and improving response times.

### Building Observability into Your Systems
Getting started with observability involves collecting the right data, setting up the appropriate tools, and fostering a culture where teams actively use and respond to the insights provided. We need to instrument our existing systems with auto instrumentation agents like OTEL agent or integrating libraries to export prometheus/otel format metrics. 

**Start Small:** Begin by implementing basic logging in your applications.
**Choose Your Tools:** There are many observability tools available, both open-source and commercial. Popular choices include Prometheus, Grafana, and Jaeger.
**Define What's Important:** Identify the key metrics and events that are most crucial for your system's health and performance.
**Implement Instrumentation:** Add code to your applications to collect and send observability data.
**Visualize Your Data:** Use dashboards to make your observability data easy to understand and act upon.
**Foster a Culture of Observability:** Encourage your team to use observability data in their daily work and decision-making processes.


### References:
1. https://copyconstruct.medium.com/monitoring-and-observability-8417d1952e1c
2. https://softwareengineeringdaily.com/2021/02/04/debunking-the-three-pillars-of-observability-myth/
3. https://thenewstack.io/observability-wont-replace-monitoring-because-it-shouldnt/
4. https://charity.wtf/category/observability/







