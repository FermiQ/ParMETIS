# libparmetis/kmetis.c

## Overview

This file is the entry point for the core parallel k-way multilevel partitioning algorithm in ParMETIS, specifically `ParMETIS_V3_PartKway`. It orchestrates the entire process of taking a distributed graph and partitioning it into `k` parts. This involves setting up control structures, handling small or trivial graph cases, and invoking the main parallel multilevel partitioning routine (`Global_Partition`).

## Key Components

### Functions/Classes/Modules

*   **`ParMETIS_V3_PartKway(idx_t *vtxdist, idx_t *xadj, ..., MPI_Comm *comm)`**:
    *   This is the main V3 API function for general-purpose parallel k-way graph partitioning.
    *   **Functionality**:
        1.  Performs input validation (`CheckInputsPartKway`).
        2.  Initializes memory management and sets up the `ctrl_t` control structure (`SetupCtrl`).
        3.  Handles trivial cases:
            *   If `nparts == 1`, assigns all local vertices to partition 0 (or 1 if `numflag == 1`).
            *   If `npes == 1` (single processor), it calls the serial METIS `METIS_PartGraphKway` to perform the partitioning.
        4.  If `numflag > 0` (1-based indexing), converts graph numbering to 0-based (`ChangeNumbering`).
        5.  Sets up the distributed graph structure (`SetupGraph`).
        6.  Allocates workspace (`AllocateWSpace`).
        7.  Sets `ctrl->CoarsenTo`, a threshold for stopping coarsening.
        8.  Decides whether to partition serially or in parallel:
            *   If the total number of vertices (`vtxdist[npes]`) is small (less than `SMALLGRAPH` or `npes*20`) or the graph has no edges globally, it calls `PartitionSmallGraph` (which likely assembles the graph and uses serial METIS).
            *   Otherwise, it calls `Global_Partition` to perform parallel multilevel partitioning.
        9.  Remaps the graph based on the computed partition (`ParallelReMapGraph`).
        10. Copies the final partition assignments from `graph->where` to the user's output `part` array.
        11. Stores the computed edge cut in `*edgecut`.
        12. Prints timing and statistics if debug flags are set.
        13. Frees graph structures and control structures.
        14. If `numflag > 0`, converts numbering back to 1-based.
        15. Checks for memory leaks.
*   **`Global_Partition(ctrl_t *ctrl, graph_t *graph)`**:
    *   This is the core recursive multilevel k-way partitioning algorithm.
    *   **Functionality (Recursive Multilevel Approach)**:
        1.  Sets up communication structures for the current graph level (`CommSetup`).
        2.  **Base Case**: If the global number of vertices (`graph->gnvtxs`) is small enough (e.g., `< 1.3 * ctrl->CoarsenTo`) or the graph is significantly smaller than its finer version (if one exists):
            *   An initial k-way partition is computed using `InitPartition` (recursive bisection based).
            *   If it's the original graph (no finer graph, i.e., `graph->finer == NULL`), it applies k-way FM refinement (`KWayFM`).
        3.  **Recursive Step**: If the graph is large enough:
            *   Coarsens the graph using `Match_Global` to create `graph->coarser`.
            *   Optionally writes graph data to disk if `ctrl->ondisk` is set (`graph_WriteToDisk`).
            *   Recursively calls `Global_Partition` on `graph->coarser`.
            *   Optionally reads graph data from disk if it was offloaded (`graph_ReadFromDisk`).
            *   Projects the partition from `graph->coarser` back to the current `graph` (`ProjectPartition`).
            *   Computes partition parameters (like `lnpwgts`, `gnpwgts`, `mincut`) for the current level using `ComputePartitionParams`.
            *   If multi-constraint and not too coarse, it may perform k-way balancing (`KWayBalance`) if imbalance is detected.
            *   Applies k-way FM refinement (`KWayFM`) to improve the projected partition.
        4.  Prints progress if debug flags are set.

## Important Variables/Constants

*   **`ctrl_t`**: The ParMETIS control structure. Key fields used: `nparts`, `ncon`, `CoarsenTo`, `dbglvl`.
*   **`graph_t`**: The ParMETIS graph structure, representing the graph at different levels of coarsening.
*   **`vtxdist`, `xadj`, `adjncy`, `vwgt`, `adjwgt`**: Standard CSR graph representation and weights.
*   **`part`**: Output array for partition assignments.
*   **`edgecut`**: Output variable for the number of edges cut.
*   **`options`**: User-provided options array.
*   **`SMALLGRAPH`**: Constant from `defs.h` to decide between serial and parallel partitioning for the whole graph.
*   **`COARSEN_FRACTION`**: Constant from `defs.h` used in the base case condition of `Global_Partition`.
*   **`NGR_PASSES`**: Constant from `defs.h` for the number of FM refinement passes.

## Usage Examples

```c
// This file implements the ParMETIS_V3_PartKway API function.
// Example of calling it:
// (Assume vtxdist, xadj, adjncy, vwgt, adjwgt, part, comm, etc. are initialized)
// idx_t ncon = 1, nparts = 4;
// idx_t options[PARMETIS_MAX_OPTIONS]; options[0] = 0; // Default options
// idx_t wgtflag = 0; // No weights initially
// idx_t numflag = 0; // 0-based indexing
// real_t ubvec[1]; ubvec[0] = 1.05;
// real_t *tpwgts = NULL; // Equal target partition weights
// idx_t edgecut_val;
//
// ParMETIS_V3_PartKway(vtxdist, xadj, adjncy, vwgt, adjwgt,
//                      &wgtflag, &numflag, &ncon, &nparts,
//                      tpwgts, ubvec, options,
//                      &edgecut_val, part, &comm);
```

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Core definitions.
    *   `ctrl.c`: For `SetupCtrl`, `FreeCtrl`.
    *   `graph.c`: For `SetupGraph`, `FreeInitialGraphAndRemap`, `graph_WriteToDisk`, `graph_ReadFromDisk`.
    *   `wspace.c`: For `AllocateWSpace`, `AllocateRefinementWorkSpace`.
    *   `comm.c`: For `CheckInputsPartKway`, `GlobalSEMinComm`, `GlobalSESum`, `CommSetup`, etc.
    *   `initpart.c`: For `InitPartition`.
    *   `serial.c`: For `PartitionSmallGraph`.
    *   `remap.c`: For `ParallelReMapGraph`.
    *   `match.c`: For `Match_Global`.
    *   `kwayrefine.c`: For `ProjectPartition`, `ComputePartitionParams`, `KWayFM`, `KWayBalance`.
    *   `util.c`: For `ChangeNumbering`, `ravg`, `rargmax_strd`.
    *   `metis.h` (external METIS library): For `METIS_PartGraphKway` when `npes == 1`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: For all parallel communication.
    *   `METIS`: Serial METIS library is called as a fallback if `npes == 1`.
*   **Other Interactions**:
    *   `ParMETIS_V3_PartKway` is the main entry point for parallel k-way partitioning in ParMETIS when no geometric information is used.
    *   The `Global_Partition` function implements the classic multilevel paradigm: Coarsen -> Initial Partition (on coarsest) -> Uncoarsen & Refine.

```
