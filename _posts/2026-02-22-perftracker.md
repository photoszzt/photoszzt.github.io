---
layout: post
title: perftracker
date: 2026-02-17T16:39:58-07:00
description:  Online Performance Troubleshooting for Large-scale Model Training in Production
tags: paper-note
categories: paper-note
giscus_comments: true
---

### PerfTracker: Online Performance Troubleshooting for Large-scale Model Training in Production [link](https://arxiv.org/pdf/2506.08528)

#### Observation

LMT function (Python functions, GPU/CPU kernel functions, memory operations, etc.) executions exhibit two significant characteristics.
- Within a single worker, most low-level functions are executed repeatedly. This is because training involves many identical iterations, and models are typically composed of repetitive submodules (e.g., transformer blocks).
- Runtime behaviors of functions are highly identical across workers, because modern parallelisms (e.g., at data, pipeline, tensor, and expert levels) distribute workloads evenly with frequent synchronization operations.
- Most performance issues can be diagnosed by observing abnormal function runtime behaviors in comparison to all other function executions

#### Insight

- (1) Performance issues can be observed by profiling the behavior of function executions
- (2) We can troubleshoot performance issues using differential observability that localizes the offending function executions with abnormal behavior (e.g., low average GPU-NIC throughput without fluctuation)
- (3) We do not need to analyze fine-grained raw observability data of all the functions; instead, we only need to summarize their runtime behavior patterns.

#### Design

<img width="1625" height="553" alt="image" src="https://github.com/user-attachments/assets/a422f84c-4d1d-40bf-8272-7a3b34ce8f12" />

- (1) detecting performance degradation of LMT to trigger online profiling (per worker). 
- (2) summarizing runtime behavior patterns of each function from raw profiling data (per worker).
- (3) a centralized localization algorithm that pinpoints the root-cause function based on the behavior patterns (global).

