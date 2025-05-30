# libparmetis/initpart.c

## Overview

This file implements an initial graph partitioning algorithm for ParMETIS based on parallel multilevel recursive bisection. The core idea is to assemble the entire graph on groups of processors and then perform a sequence of recursive bisections in parallel to achieve the desired number of partitions. This method is used to get a starting partition, which can then be refined by other algorithms.

## Key Components

### Functions/Classes/Modules

*   **`InitPartition(ctrl_t *ctrl, graph_t *graph)`**:
    *   This is the main entry point for the recursive bisection based initial partitioning.
    *   It first assembles the entire graph (`agraph`) onto subgroups of processors using `AssembleAdaptiveGraph`.
    *   Processors are split into `ngroups` (e.g., `RIP_SPLIT_FACTOR`) using `MPI_Comm_split`. Each group then independently computes a full k-way partition.
    *   **Recursive Bisection within each group**:
        *   Each processor group (communicator `ipcomm`) performs a recursive bisection process.
        *   In each step of the recursion, the current graph/subgraph (`agraph` which gets progressively smaller) is bisected using `METIS_PartGraphRecursive`. The target weights for the bisection (`tpwgts2`) are derived from the overall target weights (`ctrl->tpwgts`).
        *   Processors within the group then decide which part of the bisection to follow based on their rank (`mype`) within the group. `KeepPart` is called to prune `agraph` to only the vertices belonging to the selected part.
        *   This continues until either the number of PEs in the subgroup (`lnpes`) is 1 or the number of remaining parts to be found (`lnparts`) is 1.
        *   If `lnparts > 1` when `lnpes == 1` (meaning one PE needs to find multiple partitions for its remaining subgraph), `METIS_PartGraphKway` is called to complete the partitioning for that subgraph.
    *   The resulting partition for the entire graph (from the perspective of each group) is stored in `gwhere1`.
    *   If multiple groups (`ngroups > 1`) were used, a selection process occurs:
        *   Each group computes the edge cut and balance of its computed partition (`gwhere1`) on the original full graph.
        *   The partition that yields the overall best quality (minimum cut, or if cuts are similar, best balance) is selected using `MPI_Allreduce` with `MPI_MINLOC`.
        *   The winning partition is then broadcast (`gkMPI_Bcast`) to all processors.
    *   The final chosen partition is scattered from `gwhere1` to the local `graph->where` array on each processor.
*   **`KeepPart(ctrl_t *ctrl, graph_t *graph, idx_t *part, idx_t mypart)`**:
    *   A utility function that modifies the input `graph` structure to contain only the vertices belonging to a specified partition `mypart`.
    *   It compacts `graph->xadj`, `graph->vwgt`, `graph->adjncy`, `graph->adjwgt`, and `graph->label` by removing vertices not in `mypart`.
    *   The `rename` array is used to map old vertex indices to new compacted indices. Adjacency lists are updated with these new indices.

## Important Variables/Constants

*   **`agraph` (graph_t)**: The assembled graph structure containing the entire graph data, used by each processor group for recursive bisection.
*   **`part` (idx_t array in `InitPartition`)**: Temporary array to store partition assignments during METIS calls.
*   **`gwhere0`, `gwhere1` (idx_t arrays in `InitPartition`)**: Arrays to store the full graph partition computed by each processor group.
*   **`ctrl->tpwgts`**: Target partition weights for the overall k-way partition.
*   **`tpwgts2`**: Target weights for a 2-way bisection, derived from `ctrl->tpwgts`.
*   **`RIP_SPLIT_FACTOR`**: Constant from `defs.h` suggesting how many groups to split PEs into.
*   **`moptions`**: Array for METIS options, including seed and potentially performance-tuning flags if `ctrl->fast` is set.
*   MPI Communicators: `ipcomm` (for processor groups).

## Usage Examples

```
N/A
```
This is an internal ParMETIS function. `InitPartition` is typically called by `Global_Partition` (in `kmetis.c`) when a graph needs an initial k-way partition from scratch, especially when the graph is deemed large enough for parallel processing but no prior partitioning information is available.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides `ctrl_t`, `graph_t` definitions, MPI wrappers, memory utilities, etc.
    *   `initbalance.c`: `AssembleAdaptiveGraph` is used to gather the graph.
    *   `serial.c`: `KeepPart` (though defined in this file, it's a utility for graph manipulation), `ComputeSerialEdgeCut`, `ComputeSerialBalance`.
    *   `graph.c`: For `FreeGraph`, `icopy`.
    *   `metis.h` (external METIS library): `METIS_SetDefaultOptions`, `METIS_PartGraphRecursive`, `METIS_PartGraphKway`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Used for communication (`gkMPI_Comm_split`, `gkMPI_Comm_rank`, `gkMPI_Comm_size`, `gkMPI_Allreduce`, `gkMPI_Bcast`, `gkMPI_Comm_free`).
    *   `METIS`: The serial METIS library is used as the core engine for recursive bisection and k-way partitioning on the assembled graph data within each processor group.
*   **Other Interactions**:
    *   The function modifies `graph->where` with the computed initial partition.
    *   The quality of the initial partition generated here can significantly affect the subsequent refinement phases and the final partitioning quality.
    *   The use of multiple processor groups trying different random seeds (`ctrl->sync + (ctrl->mype % ngroups) + 1`) is a way to explore different partitioning solutions and pick the best one.

```
