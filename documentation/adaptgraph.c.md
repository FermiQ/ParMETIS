# programs/adaptgraph.c

## Overview

This file contains functions designed to simulate graph adaptation, primarily for testing ParMETIS's adaptive repartitioning routines (`ParMETIS_V3_AdaptiveRepart`). The functions modify vertex weights or edge weights of a distributed graph to simulate changes in computational load or communication patterns that might occur in a dynamic application.

## Key Components

### Functions/Classes/Modules

*   **`AdaptGraph(graph_t *graph, idx_t afactor, MPI_Comm comm)`**:
    *   Adapts the input `graph` by modifying its vertex weights.
    *   `afactor`: A factor by which selected vertex weights are multiplied.
    *   Randomly selects a number of local vertices (`nadapt`) using `FastRandomPermute`.
    *   For these selected vertices, `vwgt[perm[i]]` is multiplied by `afactor`.
    *   If `graph->adjwgt` is `NULL`, it allocates and initializes it to 1. (The commented-out section suggests an idea to also adapt edge weights based on new vertex weights, but it's not active).
    *   Prints initial load imbalance statistics after adaptation.
*   **`AdaptGraph2(graph_t *graph, idx_t afactor, MPI_Comm comm)`**:
    *   Another version of graph adaptation.
    *   With a small probability (2 out of `npes` PEs, or if `RandomInRange(npes+1) < 2`), it multiplies *all* local vertex weights on the selected PE by `afactor`. This creates a more significant, localized imbalance.
    *   The commented-out section for adapting edge weights based on vertex weights is present here as well.
    *   Prints initial load imbalance statistics.
*   **`Mc_AdaptGraph(graph_t *graph, idx_t *part, idx_t ncon, idx_t nparts, MPI_Comm comm)`**:
    *   Adapts a multi-constraint graph based on an existing partition `part`.
    *   PE 0 generates random new target weights (`pwgts`) for each partition and each constraint.
    *   These target weights are broadcast to all PEs.
    *   Each vertex `i` then gets its `vwgt[i*ncon+h]` set to `pwgts[part[i]*ncon+h]`. This means all vertices in the same partition `p` get assigned the new target weight for partition `p` for each constraint. This simulates a scenario where the ideal load per vertex changes based on its current partition.

## Important Variables/Constants

*   **`graph->vwgt`**: Vertex weights, directly modified by these functions.
*   **`graph->adjwgt`**: Edge weights, potentially initialized if NULL.
*   **`afactor`**: The multiplication factor for adapting vertex weights.
*   **`part` (in `Mc_AdaptGraph`)**: The existing partition of the graph.

## Usage Examples

```c
// These functions are intended for testing ParMETIS.
// A typical test scenario (e.g., in testadpt.c or similar):
//
// // 1. Read or generate a graph
// ParallelReadGraph(&graph, filename, comm);
//
// // 2. Partition it initially
// ParMETIS_V3_PartKway(..., &graph, ..., part_initial, ...);
//
// // 3. Adapt the graph to simulate dynamic changes
// AdaptGraph(&graph, adaptation_factor, comm); // or AdaptGraph2 / Mc_AdaptGraph
//
// // 4. Call adaptive repartitioning
// ParMETIS_V3_AdaptiveRepart(..., &graph, ..., part_adapted, ...);
//
// // 5. Analyze the quality of part_adapted
```

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetisbin.h`: Likely includes `parmetislib.h` and other common headers for ParMETIS executables/test programs. Provides `graph_t` and MPI wrappers.
    *   `parmetislib.h` (indirectly): For `graph_t` structure, `idx_t` type, `gkMPI_*` wrappers, `ismalloc`, `imalloc`, `FastRandomPermute`, `RandomInRange`, `isum`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Used for `MPI_Comm_size`, `MPI_Comm_rank`, `MPI_Allreduce`, `MPI_Bcast`.
    *   Standard C library: `stdlib.h` (for `srand`, `rand` via `RandomInRange`), `stdio.h` (for `printf`), `math.h` (for `pow` in commented sections).
*   **Other Interactions**:
    *   These functions directly modify the input `graph_t` structure, particularly its `vwgt` array.
    *   They are designed to create imbalances or changes in the graph that adaptive repartitioning algorithms should then try to resolve.
    *   The randomness ensures varied test conditions.

```
