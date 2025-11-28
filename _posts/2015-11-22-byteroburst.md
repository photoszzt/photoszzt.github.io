---
layout: post
title: ByteRobust
date: 2025-11-22T16:39:58-07:00
description: Robust LLM Training Infrastructure at ByteDance
tags: paper-review
categories: paper-review
---

### Robust LLM Training Infrastructure at ByteDance [link](https://arxiv.org/abs/2509.16293)

Control plane

1. Robust controller  
   1. Robust controller react to events from proactive realtime check:  
      1. High confidence event for a specific machine: force drain node, evict the machine, skip stop time diagnostics  
         1. GPU unavailability  
         2. Disk fault  
      2. Network issue: tolerate several alerts before evict the machine. Policy: twice within 5 mins empirically  
         1. NIC and Network switch flapping can auto recover  
      3. If the restarted job fails again after machine eviction, it goes to a stop-time check procedure.   
   2. Robust controller analyze the logs  
      1. User space error: traceable to specific code modules from logs and exit codes, python exception \-\> code rollback  
      2. Training crashes/abnormal metrics, e.g., NaN losses, arise without a clear culprit \-\> stop time check  
      3. Performance anomalies: 0 RDMA traffic within 10 mins, low TensorCore utilization \-\> aggregation analysis  
2. Runtime analyzer  
   1. Aggregation analysis  
      1. Parses process trees in each training pod to identify training related processes (torchrun, dataloader, checkpoint process)  
      2. Stack traces from these identified processes are aggregated into multiple groups via string matching to differentiate abnormal sources  
         1. The dominant groups are deemed healthy.   
         2. Remaining groups are classified as outliers.  
      3. Find shared parallel groups for those outliers and isolate the corresponding machines  
   2. Fail slow (MFU decline):  
      1. Repeat aggregation every 10s, flag the parallel group with the most outliers at each round.   
      2. Parallel group with the highest cumulative flag count across 5 rounds is marked as the degrader for over-eviction. 

Data plane (Robust agent in each training pod)

1. Monitor  
   1. System inspection (Proactive realtime check):  
      1. Network: NIC down or jitter, packet loss rate, switches down  
      2. GPU: status of DCGM service, PCIe bandwidth, memory row remapping, and GPU temperature, etc.  
      3. Host: OS kernel event (Xid in dmesg)
   2. Metics collection  
      1. Workload specific training metrics from wandb: loss, gradient norm, MFR, etc  
         1. Fatal signal: 5× increase in loss/gradient norms, NaN values  
      2. stdout/stderr logs and process exit codes, which serve as hints for diagnostics  
      3. Events: Significant declines serve as a signal for potential job hangs and MFU declines.   
         1. CUDA  
         2. RDMA: traffic  
         3. Host  
         4. Storage  
2. Diagnoser  
   1. NaN loss diagnosis  
      1. Standard GPU and network tests first (EUD and NCCL tests).   
         1. Intra machine all-to-all test to verify bandwidth  
         2. Inter machine all-gather NCCL test to verify connectivity and integrity of data transfer with “neighboring” machines.   
      2. Bitwise alignment test  
         1. Each machine initiates a reference model whose structure matches that of the target training job (dense models or MoE models).   
         2. Load predefined weights, employs a specific parallelism configuration (e.g., TP=2, PP=2, DP=2 or EP=2, PP=2, DP=2), and executes one training step on fixed input to ensure reproducibility.  
         3. The outputs from all machines are collected and analyzed to verify bit-wise accuracy.   
         4. Machines that yield incorrect results are promptly isolated and removed.  
         5. If this test does not identify any defective machines, reattempt and rollback are sequentially employed to settle potential transient failures and human errors.  
   2. Job hang and MFU declines  
3. On demand tracer  
4. CKPT manager
