---
layout: post
title: Megascale
date: 2025-12-20
description: Scaling Large Language Model Training to More Than 10,000 GPUs
tags: paper-review
categories: paper-review
giscus_comments: true
---

1. Fault-tolerant training
   - A driver process identifies the failed pod by heartbeat. Driver pauses the training, run self diagnostic test, identify faulty node, driver submits the IP addresses of the nodes to be blocked, along with the information of the Pods running on them, to

     Kubernetes evicts the faulty nodes and replenishes the cluster with an equivalent amount of healthy ones, which pass our diagnostic tests.

2. Data collection
   - IP address, the Pod name, hardware information, current status of the

     training processes, stdout/stderr logs of training processes, RDMA traffic metrics(look for significant decline or abnormal fluctuation),

3. Diagnostic Tests
4. Intra-host test
   1. The Loopback test measures the loopback bandwidth from all RDMA NICs (RNICs) to various intra-host endpoints, including memory nodes and GPUs. It conducts a full-mesh test within the host, covering all possible link combinations.
   2. RNICto-RNIC test examines the connectivity and bandwidth performance between different RNICs on the same host.
5. NCCL tests
   1. all-to-all test among the GPUs within a single node
   2. all-reduce test with neighboring machines under the same ToR switch
6. Fast checkpoint
   1. First step: each GPU worker writes its on-chip states to the host memory, and then continues the training process
   2. Second step: a background process takes over, asynchronously transferring the state from the host memory to a distributed file system (HDFS in our deployment) for centralized maintenance

7. Fast recovery:
   1. Observation: Multiple GPU workers often share the same state partition, e.g., the workers in the same data parallel group
   2. A single worker in the group to read the shared state partition from HDFS, then broadcasts the state partition to all other GPU workers that share the same data.
8. Performance Diagnosis with CUDA Event Monitor
   1. Observation: MFU for various training tasks gradually declines over time
   2. Method: records the execution time of critical code segments on each machine rank during a run
   3. Visualization:
      1. Heat map to show time consumption differences between machines from various dimensions
         1. latency data of the computation phase (forward and backward) across devices and average the latency across steps
      2. The event timeline on machines in a trace format from different distributed views (data parallelism, pipeline parallelism, tensor parallelism).
   4. Implementation:
      1. timer data is wrote to a local file in a line-by-line format,
      2. synchronizes this log file with a Kafka queue in real-time.
      3. the analytical database remains updated by consuming data from this Kafka queue
9. 3D Parallel Training Visualization
   1. Each GPU worker logs its own ongoing event upon communication timeout. These logs are then used to construct a visual representation of data dependencies based on the logical topology in the 3D parallel setting.
10. Collective Communication Group Initialization:
    1. Use Redis for initialization store instead of TCPStore
    2. Reduce global barrier
11. Data pipeline
    1. Asynchronous data preprocessing: While the GPU workers are synchronizing gradients at the end of each training step, the data preprocessing for the subsequent step can start, which hides the preprocessing overhead.
    2. Redundant data loader elimination: GPU workers within the same machine are in the same tensor parallel group. Their inputs for each iteration are inherently identical. Only need one copy of data in CPU memory instead of each GPU worker copies memory by itself.
12. Network
    1. NCCL tune retransmit timer and retry count for fast recovery (doesn’t mention what param and tune to what)
