# libparmetis/stat.c

## Overview

This file contains utility functions for computing and printing various statistics related to graph partitions in ParMETIS. These include calculating load balance (for both serial and parallel contexts), printing partition quality information (edge cut, balance), and computing statistics about vertex movements during adaptive refinement or repartitioning.

## Key Components

### Functions/Classes/Modules

*   **`ComputeSerialBalance(ctrl_t *ctrl, graph_t *graph, idx_t *where, real_t *ubvec)`**:
    *   Computes the load balance for a k-way partition of a serial graph.
    *   `where`: Array indicating partition assignment for each vertex.
    *   `ubvec` (output): Array where `ubvec[j]` will store the imbalance for the j-th constraint. Imbalance is calculated as `max_part_weight_j / ideal_part_weight_j`.
    *   `ideal_part_weight_j` is `total_weight_j / nparts`.
*   **`ComputeParallelBalance(ctrl_t *ctrl, graph_t *graph, idx_t *where, real_t *ubvec)`**:
    *   Computes the load balance for a k-way partition of a distributed graph.
    *   `where`: Local partition assignments.
    *   `ubvec` (output): Imbalance for each constraint.
    *   It first computes local partition weights (`lnpwgts`) using normalized vertex weights (`graph->nvwgt`).
    *   Then, it performs an `MPI_Allreduce` to get global partition weights (`gnpwgts`).
    *   Imbalance for constraint `j` is `max_over_i(gnpwgts[i*ncon+j] / tpwgts[i*ncon+j])`, adjusted by a minimum vertex weight to handle zero target weights.
*   **`Mc_PrintThrottleMatrix(ctrl_t *ctrl, graph_t *graph, real_t *matrix)`**:
    *   Prints a matrix (presumably `npes x npes`) representing some form of "throttle" or interaction between processors. Output is synchronized by PE.
*   **`PrintPostPartInfo(ctrl_t *ctrl, graph_t *graph, idx_t movestats)`**:
    *   Prints summary information after a partitioning or refinement process.
    *   Displays final edge cut (`graph->mincut`).
    *   Calculates and prints the balance for each constraint using global partition weights (`graph->gnpwgts`) and target weights (`ctrl->tpwgts`).
    *   If `movestats` is true, calls `Mc_ComputeMoveStatistics` and prints vertex movement data.
*   **`ComputeMoveStatistics(ctrl_t *ctrl, graph_t *graph, idx_t *nmoved, idx_t *maxin, idx_t *maxout)`**:
    *   (Slightly different version than `Mc_ComputeMoveStatistics` in `diffutil.c` or `mdiffusion.c`; this one seems simpler and counts vertices rather than vertex weights/sizes for movement).
    *   Calculates:
        *   `nmoved`: Total number of local vertices not in their assigned partition `ctrl->mype`.
        *   `maxout`: Maximum number of vertices moved out from any single PE (which is the local `j`).
        *   `maxin`: Maximum number of vertices moved into any single PE.
    *   Uses `MPI_Allreduce` and `GlobalSEMax/GlobalSESum`.

## Important Variables/Constants

*   **`graph->vwgt`**: Vertex weights (used in `ComputeSerialBalance`).
*   **`graph->nvwgt`**: Normalized vertex weights (used in `ComputeParallelBalance`).
*   **`graph->where` / `where`**: Partition assignment array.
*   **`ctrl->tpwgts`**: Target partition weights.
*   **`graph->gnpwgts`**: Global actual partition weights.
*   **`graph->mincut`**: Computed global edge cut.

## Usage Examples

```
N/A
```
These are internal utility functions.
*   `ComputeSerialBalance` and `ComputeParallelBalance` are called by various algorithms to assess the balance of current or proposed partitions.
*   `PrintPostPartInfo` is called at the end of major operations (like `ParMETIS_V3_PartKway`, `ParMETIS_V3_RefineKway`) if debugging flags for informational output are enabled.
*   `ComputeMoveStatistics` provides metrics for how much data redistribution occurred.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Core definitions (`ctrl_t`, `graph_t`), types, MPI wrappers, memory utilities.
    *   `diffutil.c` or `mdiffusion.c`: For `Mc_ComputeMoveStatistics` (though there's a local simpler version `ComputeMoveStatistics` here too).
    *   GKlib utilities: `ismalloc`, `rset`, `rwspacemalloc`, `gk_max`, `gk_free`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Used in `ComputeParallelBalance`, `Mc_PrintThrottleMatrix`, and `ComputeMoveStatistics` for collective operations.
*   **Other Interactions**:
    *   These functions provide crucial feedback on algorithm performance (balance, cut) and data movement, often used for debugging or reporting final quality.
    *   The balance calculation methods differ slightly between serial and parallel versions, mainly in how total weights and target weights are handled (raw weights vs. normalized weights).

```
