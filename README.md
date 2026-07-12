# Scaling R-tree Spatial Search on Processing-in-Memory

This repository accompanies the HPDC 2026 poster **“Challenges in Scaling R-tree Spatial Search on Processing-in-Memory.”** It studies how an irregular, output-sensitive R-tree range-search workload behaves on a commercial **UPMEM Processing-in-Memory (PIM)** system.

The work extends our ISC 2026 paper, **“Parallel R-tree-based Spatial Query Processing on a Commercial Processing-in-Memory System,”** which introduced the broadcast-based CPU-DPU R-tree design used here. The HPDC extension keeps the same core methodology and focuses on the system-level challenges that appear when the design is scaled across datasets, query volumes, and DPU counts: workload imbalance, host-DPU communication, result movement, and CPU-side aggregation.

## Repository contents

- [`HPDC_poster.pdf`](HPDC_poster.pdf) - HPDC 2026 poster on scaling challenges and full-pipeline behavior.
- [`ISC_paper.pdf`](ISC_paper.pdf) - ISC 2026 paper describing the original broadcast-based R-tree methodology, implementation, and evaluation.

> **Note:** This repository currently contains the research artifacts above. Source code and datasets are not included in this snapshot.

---

## Why this work matters

Spatial systems repeatedly evaluate questions such as:

- Which buildings overlap a geographic region?
- Which lakes intersect an area of interest?
- Which mapped objects fall inside or intersect a query window?

An R-tree reduces the number of spatial objects that must be examined by organizing them in a hierarchy of **minimum bounding rectangles (MBRs)**. However, an R-tree search is not a regular scan. The nodes visited by a query depend on the query location, size, selectivity, and overlap among tree regions. This creates irregular memory accesses, branch divergence, variable output sizes, and uneven work.

<p align="center">
  <img src="images/Irregular_Rearch.png" alt="Examples of irregular R-tree traversal" width="760">
</p>
<p align="center"><em>Different query rectangles visit different branches and produce different amounts of work.</em></p>

On a conventional CPU, these searches repeatedly move R-tree nodes and object records through the memory hierarchy. This work investigates whether PIM can reduce that movement by executing rectangle-overlap tests close to DRAM, while also identifying the remaining bottlenecks in the complete CPU-DPU pipeline.

---

## Relationship to GIS

This project is directly related to **Geographic Information Systems (GIS)** and spatial databases. GIS datasets commonly contain points, polylines, and polygons representing real-world features such as buildings, lakes, roads, parcels, and sports facilities. Spatial systems use R-trees and related indexes to accelerate:

- map-viewport and window queries;
- spatial filtering and intersection search;
- spatial joins and overlay processing;
- subset extraction and download; and
- candidate generation before exact geometry tests.

<p align="center">
  <img src="images/Rtree_GIS.png" alt="GIS objects represented by minimum bounding rectangles and organized in an R-tree" width="850">
</p>
<p align="center"><em>GIS objects are represented by MBRs and indexed hierarchically by an R-tree.</em></p>

For indexing, a complex feature is represented by its axis-aligned MBR. During a query, the R-tree prunes branches whose MBRs do not intersect the query rectangle. The present implementation accelerates this fundamental GIS filtering operation and returns overlap counts. In a complete GIS pipeline, the candidate objects can subsequently be checked with an exact polygon or polyline predicate.

The real datasets are obtained from the **UCR Spatio-temporal Active Repository (UCR-STAR)**, a public repository for large spatial datasets. UCR-STAR supports interactive map visualization, rectangular range retrieval, and spatial subset download using spatial indexes from the R-tree family [6]. This makes the evaluated operation directly representative of a core service used by spatial repositories and GIS back ends.

---

## Dataset representation and query workloads

Every indexed object and every query is represented as a two-dimensional axis-aligned rectangle:

```text
(xmin, ymin, xmax, ymax)
```

- `xmin`, `xmax`: minimum and maximum horizontal coordinates;
- `ymin`, `ymax`: minimum and maximum vertical coordinates.

### What each indexed rectangle represents

An indexed rectangle is the MBR of one source spatial feature:

| Dataset | Indexed rectangles | Meaning of one rectangle |
|---|---:|---|
| Sports | 1.7 million | MBR of one sports-related geospatial feature |
| Lakes | 8.4 million | MBR of one lake or water-body geometry |
| Buildings | 14.3 million | MBR of one building footprint |

The exact original geometry may be a polygon or another spatial feature. The rectangle stored in the R-tree is its bounding box, which is used for fast candidate filtering.

### What a query rectangle represents

A query rectangle is a spatial search window. For each query, the program counts how many indexed rectangles intersect that window. Because leaf nodes are distributed across DPUs, each DPU computes a partial count and the host sums the partial counts to obtain the final result.

Two rectangles overlap unless one is completely to the left, right, above, or below the other.

### Why query rectangles are sampled from the dataset

The experiments use approximately **1%, 5%, 10%, and 25%** of each dataset as query rectangles. Sampling queries from the same dataset provides realistic query shapes, sizes, coordinate ranges, and spatial distributions without inventing arbitrary synthetic windows. It also gives a controlled and reproducible way to increase query pressure.

The percentages fix only the **number of input queries**. They do **not** fix the number of returned overlaps. A query in a dense region may intersect many indexed objects, while a query in a sparse region may intersect only a few. Therefore, output cardinality remains data-dependent and the workload remains output-sensitive.

A sampled rectangle may intersect its corresponding indexed object, which guarantees a valid non-empty query, but the total result size is still not fixed because all additional intersections depend on local spatial density and geometry overlap. The same query sets and overlap semantics are used for CPU and PIM execution, preserving a fair comparison.

Coordinates are converted to fixed-precision 32-bit integers for DPU execution because the evaluated UPMEM DPUs are not optimized for floating-point-heavy processing.

---

## Processing-in-Memory architecture

### PIM concept

In a conventional von Neumann system, the CPU or GPU repeatedly transfers data to and from DRAM. For data-intensive workloads, this movement can dominate both runtime and energy. Processing-in-Memory moves lightweight computation closer to memory so that more operations are performed where the data reside.

<p align="center">
  <img src="images/PIM_concept.png" alt="Comparison of a conventional von Neumann architecture and a PIM architecture" width="800">
</p>
<p align="center"><em>Conceptual difference between processor-centric execution and Processing-in-Memory.</em></p>

### Server-level organization

The evaluated UPMEM server contains conventional DRAM DIMMs and PIM-enabled DIMMs attached to two host CPU sockets. A PIM-enabled DIMM contains multiple PIM chips, and each PIM chip contains multiple lightweight DRAM Processing Units (DPUs).

<table>
  <tr>
    <td align="center" width="50%">
      <img src="images/CPU_PIM.png"
           alt="Dual-socket host CPU connected to conventional DRAM and PIM-enabled memory modules"
           width="95%">
      <br>
      <em>Host CPUs orchestrate execution while PIM chips provide many near-memory DPUs.</em>
    </td>
    <td align="center" width="50%">
      <img src="images/PIM_actual.png"
           alt="Physical organization of conventional and PIM-enabled memory in the evaluated server"
           width="95%">
      <br>
      <em>Physical placement of conventional DRAM and PIM-enabled memory in the dual-socket server.</em>
    </td>
  </tr>
</table>

The evaluated hierarchy is:

```text
Host CPU sockets
  |
  +-- Conventional DRAM DIMMs
  |
  +-- UPMEM PIM DIMMs
        |
        +-- 16 PIM chips per DIMM
              |
              +-- 8 DPUs per chip
                    |
                    +-- one 64 MB MRAM bank per DPU
```

A PIM DIMM therefore provides **128 DPUs and 8 GB of PIM memory**. A 20-DIMM configuration provides a nominal total of **2,560 DPUs and 160 GB**. The experiments use up to **2,540 DPUs**, the maximum stable allocation on the evaluated system.


### Memory hierarchy inside each DPU

Each DPU contains three important memory regions:

| Memory | Capacity | Role in this project |
|---|---:|---|
| **MRAM** | 64 MB | DPU-local DRAM storing serialized leaf nodes, query batches, upper-level metadata before loading, and result buffers |
| **WRAM** | 64 KB | Fast shared scratchpad for upper-level R-tree headers, counters, control metadata, and intermediate state |
| **IRAM** | 24 KB | Instruction memory containing the DPU kernel code |

Each DPU is a lightweight in-order processor running at approximately **400 MHz** and supports multiple hardware threads called **tasklets**. The implementation uses **11 tasklets per DPU**, because performance saturates beyond that point for the studied workload.

### CPU-DPU execution model

A DPU directly accesses only its own local memory. DPUs do not communicate directly with one another, so the host CPU must orchestrate the full pipeline:

1. build and serialize the R-tree;
2. place tree partitions and query batches in DPU MRAM;
3. launch the DPU kernels;
4. retrieve partial results; and
5. aggregate the per-DPU counts.

This bulk-synchronous model rewards applications that maximize DPU-local work, reuse data across query batches, and minimize host-mediated data movement.

## Hardware challenges of using PIM

PIM reduces CPU-DRAM data movement, but it does not behave like a conventional multicore CPU or GPU. The application must be redesigned around several hardware constraints.

### 1. Small fast local memory

Only 64 KB of WRAM is available per DPU and is shared by all tasklets. A normal pointer-based R-tree cannot simply be copied into WRAM. The design must keep only compact, frequently reused metadata in WRAM and place the larger leaf-level structure in MRAM.

**Design response:** the R-tree is serialized, and only compact upper-level headers are loaded into WRAM. Leaf nodes remain in MRAM and are scanned locally.

### 2. Explicit MRAM-WRAM transfers

MRAM is much larger than WRAM but has higher access cost. The programmer must explicitly move data between them using DMA-style operations. Small, irregular accesses can be inefficient.

**Design response:** leaf nodes are laid out contiguously, transferred in contiguous slices, and processed with sequential MRAM scans after upper-level pruning.

### 3. No direct inter-DPU communication

A query can overlap spatial objects assigned to several DPUs, but DPUs cannot exchange partial results directly.

**Design response:** every query batch is broadcast to all relevant DPUs. Each DPU returns a partial per-query count, and the CPU performs the final reduction.

### 4. Host-DPU communication overhead

Kernel acceleration alone does not guarantee end-to-end acceleration. Data placement, query broadcast, kernel launch, result retrieval, and aggregation all contribute to runtime.

**Observed effect:** on the Buildings dataset with the 25% query workload, the DPU kernel takes only **1.58 s**, while CPU-side aggregation takes **22.59 s**, or **62.9%** of total execution time.

### 5. Limited per-DPU compute capability

Each DPU is intentionally lightweight. PIM is most effective for data-intensive kernels with simple arithmetic, rather than compute-heavy code requiring sophisticated cores, large caches, or high floating-point throughput.

**Design response:** coordinates are represented as 32-bit integers and the DPU kernel primarily executes simple rectangle-overlap tests.

### 6. Irregular control flow and spatial skew

R-tree traversal is query-dependent. Equal data partition sizes do not guarantee equal execution time or equal result volume. A spatially “hot” partition can receive far more intersecting queries than other partitions.

**Observed effect:** for the 25% workload, Sports reaches an **8.12x** maximum-to-mean cycle imbalance, while Lakes reaches a **29.12x** maximum-to-mean hit imbalance.

<p align="center">
  <img src="images/Query_induced_Hot%20region.png" alt="Query-induced hot region across otherwise balanced DPU partitions" width="760">
</p>
<p align="center"><em>Equal-size R-tree partitions can receive unequal work when queries concentrate in one spatial region.</em></p>

### 7. Output-sensitive result handling

The number of matches is not known in advance and varies by query and dataset. Output-heavy workloads increase result-transfer and aggregation costs, even when the search kernel itself is fast.

**Design response:** the current implementation returns per-query partial overlap counts rather than materializing every matching object. Even with this compact representation, sequential host aggregation can become the bottleneck.

### 8. Static allocation and pointer restrictions

Host virtual addresses are not meaningful on a DPU, and conventional dynamic, pointer-rich data structures are unsuitable for transfer.

**Design response:** the CPU serializes the R-tree into a flat, breadth-first array and replaces pointers with array indices.

These constraints and behaviors are consistent with prior analyses of commercial UPMEM systems, which emphasize the importance of parallel transfers, sufficient tasklet-level parallelism, contiguous MRAM access, and careful management of the small WRAM capacity [2], [3].

---

## Methodology

The HPDC work uses the **same broadcast-based R-tree methodology introduced in the ISC 2026 paper**. The HPDC extension broadens the analysis to expose scaling limitations across the full pipeline.

### Overview

```text
CPU preprocessing
  1. Load spatial rectangles
  2. Build a three-level STR R-tree
  3. Serialize the tree in breadth-first order
  4. Separate shared upper levels from leaf nodes

One-time data placement
  5. Broadcast compact root + level-1 headers to all DPUs
  6. Partition contiguous leaf-node slices across DPUs

Batched query execution
  7. Broadcast a batch of query rectangles
  8. Each DPU filters queries with shared upper-level MBRs
  9. Each DPU scans its assigned leaf slice in MRAM
 10. Each DPU returns per-query partial overlap counts
 11. The CPU retrieves and aggregates partial results
```

The implementation separates responsibilities between the host and the DPUs:

<p align="center">
  <img src="images/CPU_Side.png" alt="Host-side stages of the Broadcast PIM R-tree pipeline" width="470">
</p>
<p align="center"><em>Host side: build and serialize the R-tree, broadcast compact upper levels, and aggregate final results.</em></p>

<p align="center">
  <img src="images/DPU_side.png" alt="DPU-side stages of the Broadcast PIM R-tree pipeline" width="720">
</p>
<p align="center"><em>DPU side: store leaf partitions, process batched queries in parallel, and return sparse partial counts.</em></p>

### 1. CPU-side STR R-tree construction

The host constructs a packed R-tree using **Sort-Tile-Recursive (STR)** bulk loading. Input rectangles are ordered by their center coordinates, tiled spatially, and grouped into fixed-capacity leaf nodes. Parent nodes are constructed recursively from child MBRs.

The selected fanout and leaf capacity produce a three-level tree:

```text
Level 0: root
Level 1: internal nodes
Level 2: leaf nodes containing data rectangles
```

This keeps the shared upper-level representation small enough for DPU-local use while exposing many leaf partitions for parallel execution.

### 2. Breadth-first serialization

The pointer-based host R-tree is converted into a flat breadth-first array. The root is stored first, followed by level-1 nodes and then leaf nodes. Breadth-first ordering creates:

- a contiguous shared prefix for the upper levels; and
- a contiguous leaf region that can be partitioned across DPUs.

### 3. Compact upper-level broadcast

All DPUs need the upper-level MBRs to decide whether a query may intersect their local spatial region. Instead of transferring complete nodes, the host broadcasts compact headers containing only the metadata needed for pruning, such as node type, child count, and MBR.

### 4. Leaf partitioning

The serialized leaf level is divided into contiguous slices. Each DPU receives one slice in its 64 MB MRAM bank. The partitioning balances the number of leaf nodes, although the HPDC results show that equal data volume does not guarantee equal query-induced work.

### 5. Query batching and broadcast

Queries are processed in batches, up to 10,000 queries in the ISC implementation. Every query must be checked against the distributed leaf partitions, so the host broadcasts the query batch to the DPUs. Batching amortizes launch and transfer overhead.

### 6. Parallel DPU search

Within each DPU, queries are statically divided among tasklets. Each tasklet performs two phases:

1. **Upper-level filtering:** test the query against a small set of WRAM-resident level-1 MBRs. The DPU-index-guided mapping limits this to a bounded neighborhood.
2. **Local leaf scan:** when filtering succeeds, scan the DPU's assigned leaf nodes in MRAM and count rectangle intersections.

The leaf data are read-only, so tasklets can process disjoint query subsets without synchronization during the search.

### 7. Result retrieval and aggregation

Each DPU produces a partial count for every query. The host retrieves the partial arrays and sums them across DPUs to obtain the final overlap count for each query.

The ISC paper established that the broadcast-based layout is substantially more communication-efficient than assigning a separate serialized subtree to every DPU. The HPDC extension shows that after the search kernel becomes fast, result handling and aggregation can dominate the overall execution time.

---

## ISC paper versus HPDC extension

| Aspect | ISC 2026 paper | HPDC 2026 extension |
|---|---|---|
| Main goal | Introduce the first broadcast-based R-tree range-search implementation on commercial UPMEM PIM | Analyze why scaling the same design is difficult at the full-system level |
| Core method | CPU-built STR R-tree, BFS serialization, upper-level broadcast, distributed leaf partitions, batched DPU search | Same methodology |
| Main comparison | Broadcast-based PIM R-tree versus subtree-based PIM baseline and CPU baselines | Kernel versus end-to-end behavior across datasets, query volumes, and DPU counts |
| Primary findings | Broadcast avoids communication-dominated subtree transfer and enables scalable, energy-efficient search | DPU search scales well, but host aggregation, result movement, and query-induced imbalance limit end-to-end scaling |
| Additional dataset emphasis | Sports, Lakes, and synthetic rectangles | Sports, Lakes, and Buildings |
| New analysis | Performance, scalability, communication, and energy | Per-DPU cycle/hit imbalance, DPU-count scaling, and detailed pipeline breakdown |

---

## Selected HPDC results

Using 2,540 DPUs and 11 tasklets per DPU:

| Dataset | Query workload | CPU search | DPU kernel | Total PIM pipeline | Kernel speedup | End-to-end speedup |
|---|---:|---:|---:|---:|---:|---:|
| Sports | 25% | 23.60 s | 1.14 s | 6.14 s | 20.67x | 3.84x |
| Lakes | 25% | 316.25 s | 7.50 s | 28.01 s | 42.19x | 11.29x |
| Buildings | 25% | 114.65 s | 1.58 s | 35.89 s | 72.59x | 3.19x |

Key observations:

- DPU-side kernel speedup ranges from approximately **20x to 73x**.
- End-to-end speedup ranges from **0.87x to 11.29x**, depending on the dataset and query volume.
- Increasing the number of DPUs strongly improves kernel time, but end-to-end improvement eventually saturates.
- Sequential CPU-side aggregation becomes dominant for output-heavy workloads.
- Spatial skew creates hot DPU-owned regions even when the leaf nodes are evenly partitioned.

---

## Current limitations and future work

The current system exposes several opportunities for improvement:

1. **Parallel host aggregation** - replace sequential reduction with a multithreaded or vectorized host implementation.
2. **DPU-side partial reduction** - reduce the amount of result data returned to the CPU.
3. **Query-aware partitioning** - partition or schedule work using both data distribution and query frequency.
4. **Hot-region replication** - replicate frequently accessed spatial regions across several DPUs.
5. **Adaptive batching** - choose batch sizes based on output selectivity and communication cost.
6. **Exact geometry refinement** - extend the pipeline beyond MBR filtering to full GIS predicates where appropriate.
7. **Deeper-tree support** - develop hierarchical layouts that remain efficient when additional internal levels no longer fit in WRAM.

---

## How to cite

### ISC 2026 paper

```bibtex
@inproceedings{jannat2026parallel,
  title     = {Parallel R-tree-based Spatial Query Processing on a Commercial Processing-in-Memory System},
  author    = {Jannat, Tasmia and Gowanlock, Michael and Puri, Satish},
  booktitle = {International Supercomputing Conference (ISC)},
  year      = {2026}
}
```

Preprint: https://arxiv.org/abs/2604.14445

### HPDC 2026 poster

```bibtex
@inproceedings{jannat2026challenges,
  title     = {Challenges in Scaling R-tree Spatial Search on Processing-in-Memory},
  author    = {Jannat, Tasmia and Gowanlock, Michael and Puri, Satish},
  booktitle = {Proceedings of the International Symposium on High-Performance Parallel and Distributed Computing (HPDC)},
  year      = {2026},
  note      = {Research poster}
}
```

---

## References

1. T. Jannat, M. Gowanlock, and S. Puri, “Parallel R-tree-based Spatial Query Processing on a Commercial Processing-in-Memory System,” ISC 2026; arXiv:2604.14445. https://arxiv.org/abs/2604.14445
2. J. Gómez-Luna, I. El Hajj, I. Fernandez, C. Giannoula, G. F. Oliveira, and O. Mutlu, “Benchmarking a New Paradigm: Experimental Analysis and Characterization of a Real Processing-in-Memory System,” *IEEE Access*, vol. 10, 2022. https://doi.org/10.1109/ACCESS.2022.3174101
3. I. El Hajj et al., “PrIM: A Benchmark Suite for Processing-in-Memory,” *IEEE Transactions on Parallel and Distributed Systems*, vol. 32, no. 12, 2021. https://doi.org/10.1109/TPDS.2021.3085791
4. A. Guttman, “R-trees: A Dynamic Index Structure for Spatial Searching,” *Proceedings of ACM SIGMOD*, 1984. https://doi.org/10.1145/971697.602266
5. S. T. Leutenegger, M. A. Lopez, and J. Edgington, “STR: A Simple and Efficient Algorithm for R-tree Packing,” *Proceedings of ICDE*, 1997. https://doi.org/10.1109/ICDE.1997.582015
6. S. Ghosh, T. Vu, M. A. Eskandari, and A. Eldawy, “UCR-STAR: The UCR Spatio-Temporal Active Repository,” *SIGSPATIAL Special*, vol. 11, no. 2, 2019. https://doi.org/10.1145/3377000.3377005
7. UCR-STAR spatial dataset repository. https://star.cs.ucr.edu/

---

## Authors

- **Tasmia Jannat**, Missouri University of Science and Technology
- **Michael Gowanlock**, Northern Arizona University
- **Satish Puri**, Missouri University of Science and Technology

## Acknowledgments

This work is supported in part by the National Science Foundation grants acknowledged in the accompanying paper and poster.
