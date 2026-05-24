---
layout: post
title: perftracker
date: 2026-02-17T16:39:58-07:00
description: Online Performance Troubleshooting for Large-scale Model Training in Production
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

##### Detecting Performance Degradation

- Indicators of iteration time: A PyTorch training iteration always involves several dataloader.next() calls, followed by several optimizer.step() calls (the number depends on training parameters like pipeline parallelism). The duration from the first dataloader.next() to the last optimizer.step() is regarded as the duration of a complete training iteration.
- After detecting 𝑀 (=10 in practice) identical sequences starting with dataloader.next() and ending with optimizer.step(), this sequence is defined as the training iteration sequence.
- Performance degradation detection
  - (1) The average duration of the recent 𝑁 (=50 in practice) iterations exceeds the recent shortest iteration time by more than 5%.
  - (2) The current training iteration sequence has not yet been fully matched, but the time elapsed since the last event received is at least 5× the average iteration duration (indicating the training is blocked)
    - If PerfTracker fails to match a training iteration after 𝐾 (=200 in practice) consecutive event receptions, it goes back to the previous iteration detection phase to redetect the training iteration sequence.
- Profile generation (default 20s)
  - Torch Profiler: function execution events for Python functions, CPU operations, memory operations, and CUDA kernels
  - nsys to sample hardware metrics at 10 kHz, such as GPU, DRAM, NVLink, PCIe, and the network.

##### Summarization

- (1) finding the function execution events on the critical path, including GPU computation kernels, collective communication functions, memory operations, Python functions, and all other functions executed in LMT
- (2) clustering all execution events of each function (for Python functions, the entire call stack must be identical to be considered the same function), then defining several patterns to summarize the behavior of each function

##### Localization

Performance issues

- (1) a common problem of all LMT workers, like hardware misconfigurations and low-efficiency code implementation
- (2) a special problem on only a part of the workers, like hardware issues
