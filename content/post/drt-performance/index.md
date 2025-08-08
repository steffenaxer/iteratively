---
title: 'Fast & Curious: How to Speed Up DRT Simulations in MATSim'
description: Massive Speedup for DRT Simulations with Parallel Request Insertion
slug: drt-speedup
date: 2025-08-07 00:00:00+0000
image: cover.jpg
#categories:
#    - Example Category
tags:
   - Performance
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## 💡 Motivation

During the development of **DSim (Distributed MATSim)** — as presented at [MUM 2025](https://matsim.org/conferences/mum2025.html) — it became evident that **Demand Responsive Transport (DRT)** systems pose a significant bottleneck in large-scale MATSim simulations. Especially when simulating thousands of autonomous rides, the insertion logic for unplanned requests becomes computationally expensive and limits overall scalability.

This is problematic because **fast and scalable DRT simulations** are crucial for system planning in **Autonomous Driving Mobility-as-a-Service (AD MaaS)**. As fleets and demand grow, cities and operators need to regularly optimize service configurations. This requires simulation frameworks that can handle high request volumes efficiently.

To address this, I developed the **ParallelUnplannedRequestInserter**, a multi-threaded insertion engine designed to scale with demand and fleet size.

---

## ⚙️ Conceptual Design

The `ParallelUnplannedRequestInserter` is a parallelized implementation of MATSim’s `UnplannedRequestInserter` interface. It introduces the following key concepts:

- **Partitioning**: Incoming DRT requests and available vehicle entries are partitioned using `RequestsPartitioner` and `VehicleEntryPartitioner`.
- **Worker Threads**: Each partition is processed by a dedicated `RequestInsertWorker`, which performs insertion logic independently.
- **Conflict Resolution**: After parallel processing, insertions are consolidated. Conflicts (e.g., multiple requests assigned to the same vehicle) are resolved iteratively.
- **Thread Activity Logging**: Optionally logs request density and active partitions over time for performance diagnostics.

This design enables **concurrent evaluation of insertion possibilities**, significantly reducing simulation time for high-volume DRT scenarios.

### 🧩 Partitioning Strategies Explained

This simulation uses two interfaced partitioning strategies to improve scalability and responsiveness in high-load DRT scenarios:

#### 📥 Request-Partitioner

This partitioner (`LoadAwareRoundRobinRequestsPartitioner`) distributes incoming DRT requests across multiple partitions using a **load-aware round-robin** strategy. Unlike a static round-robin, it dynamically adjusts the number of active partitions based on the current request load and a configurable scaling function.

- For every additional **20 requests per minute**, one additional partition is activated.
- Requests are assigned cyclically to active partitions using an internal counter.
- This ensures efficient use of resources and better parallelization under varying demand levels.
- **Importantly**, under **low demand**, the partitioner avoids unnecessary fragmentation of requests. This helps maintain simulation quality by reducing empty trips and rejection rates, which would otherwise increase due to overly distributed demand.

#### 🚗 Vehicle-Partitioner

This partitioner (`ShiftingRoundRobinVehicleEntryPartitioner`) assigns vehicles to request partitions using a **shifting round-robin** strategy:

- Vehicles are distributed cyclically across active partitions.
- The starting index shifts with each invocation to avoid systematic bias.
- Only partitions with at least one request are considered.
- Vehicles are sorted deterministically by ID to ensure reproducibility.

Together, these strategies ensure that both requests and vehicles are evenly and dynamically distributed across processing threads, improving simulation performance and fairness over time.

---

## 📊 Benchmark Results

{{< figure src="benchmark.png" alt="Simulation Duration Comparison: Baseline vs Optimized" width="600" >}}

The chart compares simulation durations for two partitioning strategies across different agent counts:

- **Baseline** (orange): shows a steep increase in simulation time as the number of agents grows.
- **Optimized** (blue): maintains significantly lower durations, scaling more efficiently.

At **100,000 agents**, the optimized setup is **1.60× faster** than the baseline.  
At **200,000 agents**, the speedup increases to **2.60×**.  
An extrapolation suggests that at **400,000 agents**, the optimized strategy could be **4.60× faster**, highlighting its scalability advantage.

This benchmark demonstrates how intelligent partitioning can drastically reduce simulation time in large-scale DRT scenarios.

---

## 🛠️ How to Use

To enable the `ParallelUnplannedRequestInserter` in your MATSim DRT simulation, follow these steps:

1. **Add `DrtParallelInserterParams`** to your DRT config:
    ```java
    DrtParallelInserterParams params = new DrtParallelInserterParams();
    params.setLogThreadActivity(true); // Optional: enables logging
    drtCfg.addParameterSet(params);
    ```

2. **Replace the default inserter module** with the parallel version:
    ```java
    controller.addOverridingQSimModule(new ParallelRequestInserterModule(drtCfg));
    ```

3. **Choose your insertion strategy**:
   - For **Extensive Search**:
       ```java
       drtCfg.removeParameterSet(drtCfg.getDrtInsertionSearchParams());
       drtCfg.addParameterSet(new ExtensiveInsertionSearchParams());
       ```
   - For **Repeated Selective Search**:
       ```java
       drtCfg.removeParameterSet(drtCfg.getDrtInsertionSearchParams());
       drtCfg.addParameterSet(new RepeatedSelectiveInsertionSearchParams());
       ```

4. **Run your simulation** as usual:
    ```java
    Controler controller = DrtControlerCreator.createControler(config, false);
    controller.run();
    ```

### ⚙️ Configuration Parameters

<div style="font-size: 0.85em">

| Parameter | Description | Default |
|----------|-------------|---------|
| `collectionPeriod` | Time window (in seconds) for collecting incoming requests before processing begins. Larger values increase batching but may delay responsiveness. | `15.0` |
| `maxIterations` | Maximum number of conflict resolution iterations. Helps resolve cases where multiple partitions assign requests to the same vehicle. | `2` |
| `maxPartitions` | Maximum number of partitions (i.e., parallel workers). Each partition processes a subset of requests and vehicles. | `4` |
| `insertionSearchThreadsPerWorker` | Number of threads allocated per worker for performing insertion searches. | `4` |
| `logThreadActivity` | Enables logging of thread activity and request density. Useful for diagnostics and benchmarking. | `false` |
| `vehiclesPartitioner` | Strategy for partitioning vehicles across workers. Options:  

</div>

These parameters allow you to balance performance, parallelism, and conflict resolution quality depending on your scenario scale and hardware capabilities.

For a complete example, refer to the [`RunDrtExampleIT.java`](https://github.com/matsim-org/matsim-libs/blob/main/contribs/drt/src/test/java/org/matsim/contrib/drt/run/examples/RunDrtExampleIT.java#L88) integration test, which demonstrates various configurations.


## 🧠 Conclusion

The `ParallelUnplannedRequestInserter` is a powerful enhancement for MATSim’s DRT module, enabling **scalable, high-performance simulations** that are essential for future mobility planning. Whether you're simulating autonomous fleets or optimizing MaaS strategies, this tool ensures that your simulations remain **fast, reliable, and adaptable** — even at large scale.

By leveraging **parallel processing** and **dynamic partitioning**, it significantly improves the **economic feasibility of large-scale studies**. Simulations that previously required hours or days can now deliver actionable insights in a fraction of the time. This acceleration not only reduces computational costs but also shortens the time to decision-making — a critical factor in agile urban planning and transport strategy development.

A logical next step is to **combine these optimizations with DSim (Distributed MATSim)**, enabling distributed execution across multiple machines or clusters. This will unlock even greater scalability, allowing researchers and planners to simulate entire regions or countries with high fidelity and responsiveness.

---
<div align="center">Stay tuned for the next iteration 🔁</div>

*Author: Steffen Axer*

