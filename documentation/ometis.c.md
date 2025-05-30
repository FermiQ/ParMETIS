# libparmetis/ometis.c

## Overview

This file implements the parallel nested dissection ordering algorithm in ParMETIS. The primary API function is `ParMETIS_V3_NodeND` (and its tunable version `ParMETIS_V32_NodeND`). Nested dissection is a graph algorithm used to find a fill-reducing ordering for sparse matrix factorization. It recursively bisects (or multisects) the graph, ordering separator vertices last. This file orchestrates the multilevel version of this process.

## Key Components

### Functions/Classes/Modules

*   **`ParMETIS_V3_NodeND(idx_t *vtxdist, ..., MPI_Comm *comm)`**:
    *   The main V3 API function for parallel node nested dissection.
    *   Performs input checks (`CheckInputsNodeND`).
    *   Calls `ParMETIS_V32_NodeND` with default or derived parameters for matching type, refinement type, etc.
*   **`ParMETIS_V32_NodeND(idx_t *vtxdist, ..., idx_t *order, idx_t *sizes, MPI_Comm *comm)`**:
    *   The tunable version of the parallel node nested dissection algorithm.
    *   **Steps**:
        1.  **Initial Partitioning for Distribution**:
            *   Sets up a `ctrl_t` structure (initially for `PARMETIS_OP_KMETIS`).
            *   Calls `Global_Partition` to get an initial `k`-way partition (where `k` is often `5*npes`). This is primarily to distribute the graph reasonably for subsequent ordering steps, not the final ordering itself.
            *   The resulting `graph->where` is collapsed to `npes` partitions.
        2.  **Move Graph**: Calls `MoveGraph` to redistribute the graph according to this `npes`-way partition. The moved graph is `mgraph`.
        3.  **Multilevel Ordering on Moved Graph**:
            *   Sets `ctrl->optype = PARMETIS_OP_OMETIS`.
            *   Sets parameters like `mtype` (matching type), `rtype` (refinement type for separators), `p_nseps`/`s_nseps` (number of separators to try), `ubfrac`.
            *   Calls `MultilevelOrder` on `mgraph` to compute the actual nested dissection ordering. The ordering is stored in `morder`, and separator sizes in `sizes`.
        4.  **Project Ordering Back**: Projects `morder` from `mgraph`'s distribution back to the original graph's distribution using `ProjectInfoBack`, storing it in `order`.
        5.  Frees structures and performs cleanup.
*   **`MultilevelOrder(ctrl_t *ctrl, graph_t *graph, idx_t *order, idx_t *sizes)`**:
    *   The core multilevel nested dissection driver.
    *   `npes` is effectively the number of "parts" in the recursive dissection (must be power of 2).
    *   Initializes `perm` (tracks vertex identity through compaction) and `lastnode` (tracks available numbering range for separators).
    *   Iteratively calls `Order_Partition_Multiple` for `nparts = 2, 4, ..., npes`. In each iteration:
        *   `Order_Partition_Multiple` finds separators that divide current partitions.
        *   `LabelSeparators` assigns order numbers to the newly found separator vertices (from high to low).
        *   `CompactGraph` removes the separator vertices, creating a new graph of disconnected components for the next level of dissection.
    *   After all separator levels are found:
        *   The remaining disconnected components (now distributed one per PE that is part of the `npes` active set) are ordered locally using `LocalNDOrder`.
        *   The local orderings are projected back.
*   **`Order_Partition_Multiple(ctrl_t *ctrl, graph_t *graph)`**:
    *   Computes node separators for the current `graph` to achieve `ctrl->nparts` way separation.
    *   It performs `ctrl->p_nseps` iterations of calling `Order_Partition`.
    *   In each iteration, it tries to find a better set of separators than the previous one. The `bestwhere` and `bestseps` arrays store the best separator found so far for each bisection.
    *   After iterations, it finalizes `graph->where` with the overall best separators and computes node partition params.
*   **`Order_Partition(ctrl_t *ctrl, graph_t *graph, idx_t *nlevels, idx_t clevel)`**:
    *   A recursive multilevel function to find separators for the current `graph`.
    *   **Base Case**: If graph is small enough (`graph->gnvtxs < 1.66*ctrl->CoarsenTo`) or coarsening hasn't reduced size much:
        *   Calls `InitMultisection` to compute separators on this coarsest graph.
        *   If it's the original graph (no coarsening happened), calls node-based refinement (`KWayNodeRefine_Greedy` or `KWayNodeRefine2Phase`).
    *   **Recursive Step**:
        *   Coarsens graph using `Match_Local` or `Match_Global`.
        *   Recursively calls `Order_Partition` on `graph->coarser`.
        *   Projects partition from coarser graph (`ProjectPartition`).
        *   Refines the projected separators using `KWayNodeRefine_Greedy` or `KWayNodeRefine2Phase`.
*   **`LabelSeparators(ctrl_t *ctrl, graph_t *graph, idx_t *lastnode, idx_t *perm, idx_t *order, idx_t *sizes)`**:
    *   Assigns global order numbers to vertices identified as separators in `graph->where`.
    *   Separator vertices are numbered from `lastnode[sid] - 1` downwards.
    *   `lastnode` keeps track of the next available global order number for each separator tree branch.
    *   `sizes` array is populated with the actual sizes of these separators.
*   **`CompactGraph(ctrl_t *ctrl, graph_t *graph, idx_t *perm)`**:
    *   Removes separator vertices (those with `where[i] >= nparts`) from `graph`.
    *   The remaining disconnected components (vertices with `where[i] < nparts`) form the graph for the next level of nested dissection.
    *   Updates `graph->vtxdist`, `graph->nvtxs`, `graph->nedges`, `graph->gnvtxs`, and `graph->where` (re-labeling components `0` to `nparts-1`).
    *   `perm` array is updated to track original vertex indices through compaction.
*   **`LocalNDOrder(ctrl_t *ctrl, graph_t *graph, idx_t *order, idx_t firstnode)`**:
    *   Called on each PE once the graph is fully dissected into `npes` components, each local to a PE.
    *   Performs serial nested dissection ordering (`METIS_NodeND`) on the local graph component.
    *   The local ordering is assigned global order numbers starting from `firstnode`.

## Important Variables/Constants

*   **`order`**: Output array storing the fill-reducing permutation.
*   **`sizes`**: Output array, `sizes[0...npes-1]` stores sizes of dissected subdomains, `sizes[npes...2*npes-2]` stores separator sizes.
*   **`graph->where`**: Used dynamically to store current partition (0/1 for bisection) or separator assignments (>= `nparts`).
*   **`perm` (in `MultilevelOrder`, `CompactGraph`)**: Tracks original vertex indices across graph compactions.
*   **`lastnode` (in `MultilevelOrder`, `LabelSeparators`)**: Tracks the highest available order number for different branches of the separator tree.
*   `ctrl->mtype`, `ctrl->rtype`, `ctrl->p_nseps`, `ctrl->s_nseps`, `ctrl->ubfrac`: Control parameters for matching, refinement, and balance.

## Usage Examples
This file implements API functions `ParMETIS_V3_NodeND` and `ParMETIS_V32_NodeND`.
```c
// Conceptual call to ParMETIS_V3_NodeND
// (assuming vtxdist, xadj, adjncy, order, sizes, comm are set up)
// idx_t numflag = 0;
// idx_t options[METIS_NOPTIONS]; options[0] = 0; // Default options
//
// ParMETIS_V3_NodeND(vtxdist, xadj, adjncy, &numflag, options, order, sizes, &comm);
//
// // 'order' now contains the fill-reducing permutation.
// // 'sizes' contains the sizes of the separators.
```

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Core definitions.
    *   `ctrl.c`, `graph.c`, `comm.c`, `wspace.c`, `move.c`, `match.c`, `kwayrefine.c` (for `ProjectPartition`), `node_refine.c`, `initmsection.c`, `util.c`.
    *   `kmetis.c`: `Global_Partition` is used for initial graph distribution.
    *   `metis.h` (external METIS library): `METIS_NodeNDP` (in `pspases.c` context, but `METIS_NodeND` here) for local ordering.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`.
    *   `METIS` (for local ordering).
*   **Other Interactions**:
    *   This is a complex orchestration of coarsening, partitioning (bisection/multisection), separator refinement, and graph compaction to achieve a nested dissection ordering.
    *   The `sizes` array provides information about the structure of the separator tree.

```
