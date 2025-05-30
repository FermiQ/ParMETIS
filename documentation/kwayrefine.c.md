# libparmetis/kwayrefine.c

## Overview

This file contains core routines for k-way graph partitioning refinement and balancing in ParMETIS. It includes functions to project a partition from a coarser graph to a finer graph, compute essential parameters for refinement (like internal/external degrees and partition weights), and perform k-way refinement using a Fiduccia-Mattheyses (FM)-like approach. It also provides a k-way balancing routine.

## Key Components

### Functions/Classes/Modules

*   **`ProjectPartition(ctrl_t *ctrl, graph_t *graph)`**:
    *   Projects a partition from a coarser graph (`graph->coarser`) to the current (finer) `graph`.
    *   The `graph->match` and `graph->cmap` arrays (computed during coarsening) are used to map coarse graph vertex partitions back to the fine graph vertices.
    *   Handles communication of partition assignments for vertices whose coarse representation resides on a different processor (`PARMETIS_MTYPE_GLOBAL` matching).
    *   Frees the coarser graph structure (`graph->coarser`) after projection.
*   **`ComputePartitionParams(ctrl_t *ctrl, graph_t *graph)`**:
    *   Computes various parameters essential for refinement after a partition (`graph->where`) is established or projected.
    *   Initializes `graph->ckrinfo` (stores internal/external degrees and neighbor partition info for each vertex).
    *   Computes local (`graph->lnpwgts`) and global (`graph->gnpwgts`) partition weights for each constraint.
    *   Calculates the local (`graph->lmincut`) and global (`graph->mincut`) edge cut.
    *   Requires communication (`CommInterfaceData`) to get `where` information for ghost vertices.
*   **`KWayFM(ctrl_t *ctrl, graph_t *graph, idx_t npasses)`**:
    *   Performs k-way partitioning refinement using an FM-like iterative approach for a specified number of `npasses`.
    *   Aims to reduce edge cut while respecting balance constraints.
    *   **Algorithm Steps (per pass, per sub-iteration `c` for side preference)**:
        1.  Randomly permutes vertices (`perm`) and partitions (`pperm`).
        2.  Checks current balance (`ComputeParallelBalance`).
        3.  **Pass 1 (Propose Moves)**: Iterates through vertices. For each eligible border vertex:
            *   Identifies potential moves to neighboring partitions that improve gain (edge cut reduction) and satisfy balance constraints (`badmaxpwgt`, `IsHBalanceBetterTT`, `IsHBalanceBetterFT`). The `ProperSide` macro influences candidate target partitions.
            *   If a good move is found, updates `tmp_where` (temporary partition), local/global partition weights (`lnpwgts`, `gnpwgts`, `movewgts`), and internal/external degrees of the moved vertex and its neighbors.
        4.  Global reduction of `lnpwgts` to get proposed global weights `pgnpwgts`.
        5.  Computes `overfill` array: if proposed moves lead to imbalance beyond `badmaxpwgt`, calculates how much each partition is overfilled relative to the moves made to it.
        6.  **Undo Moves (if overweight)**: If `overweight` is detected, iterates through moved vertices and reverts moves that contribute significantly to imbalance (e.g., `overfill > nvwgt[h]/4.0`), updating weights and degrees.
        7.  **Pass 2 (Commit Moves & Update)**:
            *   Commits valid moves from `tmp_where` to `graph->where`.
            *   Identifies vertices whose `ckrinfo` needs updating locally (`update` array) or on other PEs (`supdate` array).
            *   Communicates changed interface `where` data (`CommChangedInterfaceData`) and lists of vertices needing updates (`rupdate`) to neighbors.
            *   Updates `ckrinfo` (id, ed, neighbor list) for all vertices in `update` and received in `rupdate`.
            *   Updates global partition weights `gnpwgts` and `graph->mincut`.
        8.  If cut doesn't improve, may break early from passes.
*   **`KWayBalance(ctrl_t *ctrl, graph_t *graph, idx_t npasses)`**:
    *   A k-way balancing algorithm, structurally similar to `KWayFM` but primarily focused on improving load balance rather than just edge cut.
    *   It iterates through vertices and considers moves to partitions that improve balance (`IsHBalanceBetterFT`, `IsHBalanceBetterTT`), even if the edge cut gain is not strictly positive.
    *   It does *not* have the "overfill" check and rollback mechanism present in `KWayFM`. Moves are more greedily applied if they improve balance.
    *   The selection of moves is also simpler, prioritizing balance improvement.
    *   Only a fraction of potential moves (e.g., `iii % npes == 0`) might be committed in each sub-iteration to allow for more gradual balancing.

### Macros
*   **`ProperSide(c, from, other)`**:
    *   Used in `KWayFM` and `KWayBalance` to guide move selection. `c` (0 or 1) defines a "side" preference based on a permutation of partitions (`pperm`). A move from partition `from` to `other` is considered "proper" if it aligns with the current side preference (e.g., moving from a higher permuted index to a lower one, or vice-versa).

## Important Variables/Constants

*   **`graph->where`**: Array storing the partition assignment for each vertex. Modified by refinement/balancing.
*   **`graph->match`, `graph->cmap`**: Used by `ProjectPartition`.
*   **`graph->ckrinfo` (array of `ckrinfo_t`)**: Stores refinement info: `id` (internal degree), `ed` (external degree), `inbr` (index to neighbor list), `nnbrs` (number of distinct neighbor partitions).
*   **`ctrl->cnbrpool` (array of `cnbr_t`)**: Pool of memory for storing neighbor partition info (`pid`, `ed`).
*   **`graph->lnpwgts`, `graph->gnpwgts`**: Local and global partition weights.
*   **`graph->mincut`, `graph->lmincut`**: Global and local edge cut.
*   **`ctrl->ubvec`, `ctrl->tpwgts`**: Target imbalance and target partition weights.
*   **`badmaxpwgt`**: Calculated maximum allowed weight for any partition.
*   **`perm`, `pperm`**: Permutation arrays for randomizing vertex and partition processing.
*   **`tmp_where`, `moved`, `update`, `supdate`, `rupdate`, `htable`**: Temporary arrays for managing moves and updates during refinement.
*   `NGR_PASSES`: Default number of passes for KWayFM (from `defs.h`).

## Usage Examples

```
N/A
```
These are internal ParMETIS functions.
*   `ProjectPartition` is called during the uncoarsening phase of multilevel algorithms.
*   `ComputePartitionParams` is called after projection or initial partitioning to prepare for refinement.
*   `KWayFM` is the primary k-way refinement algorithm used to improve partition quality.
*   `KWayBalance` is used when load balance is a primary concern and needs to be actively improved.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Core definitions, types, constants, MPI wrappers, memory utilities.
    *   `graph.c`: `FreeGraph` (for coarser graph), `cnbrpoolGetNext`.
    *   `comm.c`: `CommInterfaceData`, `CommChangedInterfaceData`, `GlobalSESum`.
    *   `util.c`: `RandomPermute`, `FastRandomPermute`, `ComputeParallelBalance`, `ravg`, `IsHBalanceBetterTT`, `IsHBalanceBetterFT`.
    *   `wspace.c`: For workspace allocation.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: For communication (`gkMPI_Irecv`, `gkMPI_Isend`, `gkMPI_Wait`, `gkMPI_Waitall`, `gkMPI_Bcast`, `gkMPI_Allreduce`, `gkMPI_Get_count`).
*   **Other Interactions**:
    *   These routines iteratively modify `graph->where` and associated partition parameters.
    *   The effectiveness of refinement depends on the quality of the initial partition and the number of passes.
    *   `KWayBalance` and `KWayFM` share a lot of structural similarity but differ in their primary objectives (balance vs. cut) and detailed move selection criteria.

```
