# libparmetis/ametis.c

## Overview

This file serves as the entry point for ParMETIS's parallel adaptive repartitioning capabilities. It implements a multilevel algorithm that aims to improve an existing partition or create a new one by utilizing diffusive repartitioning techniques. The core idea is to iteratively move data (vertices) across processors to minimize edge cut while maintaining or improving load balance, often involving coarsening the graph, partitioning the coarser graph, and then projecting the partition back and refining it. This is particularly useful for applications where the computational load or data distribution changes dynamically.

## Key Components

### Functions/Classes/Modules

*   **`ParMETIS_V3_AdaptiveRepart(idx_t *vtxdist, idx_t *xadj, idx_t *adjncy, idx_t *vwgt, idx_t *vsize, idx_t *adjwgt, idx_t *wgtflag, idx_t *numflag, idx_t *ncon, idx_t *nparts, real_t *tpwgts, real_t *ubvec, real_t *ipc2redist, idx_t *options, idx_t *edgecut, idx_t *part, MPI_Comm *comm)`**:
    *   This is the primary public API function in this file for performing adaptive repartitioning.
    *   **Parameters**:
        *   `vtxdist`, `xadj`, `adjncy`: Distributed graph structure (CSR format).
        *   `vwgt`: Vertex weights (if `wgtflag` indicates).
        *   `vsize`: Vertex sizes (used for redistribution).
        *   `adjwgt`: Edge weights (if `wgtflag` indicates).
        *   `wgtflag`: Flags indicating presence of vertex/edge weights.
        *   `numflag`: Flag indicating 0-based or 1-based indexing.
        *   `ncon`: Number of constraints on vertex weights.
        *   `nparts`: Desired number of partitions.
        *   `tpwgts`: Target partition weights for each constraint.
        *   `ubvec`: Load imbalance tolerance for each constraint.
        *   `ipc2redist`: A real value, likely `ipc_factor`, that balances inter-processor communication (edge cut) reduction against data redistribution.
        *   `options`: Array for various ParMETIS options (e.g., debug level, specific strategy flags).
        *   `edgecut` (output): Stores the resulting edge cut.
        *   `part` (input/output): Initially contains the existing partition (if any) and stores the new partition.
        *   `comm`: MPI communicator.
    *   **Functionality**:
        1.  Performs input validation (`CheckInputsAdaptiveRepart`).
        2.  Initializes memory management (`gk_malloc_init`).
        3.  Sets up a control structure (`ctrl_t`) with options and parameters (`SetupCtrl`).
        4.  Handles the trivial case of `nparts == 1`.
        5.  Adjusts numbering if `numflag` indicates 1-based indexing (`ChangeNumbering`).
        6.  Sets up the graph structure (`SetupGraph`), including local vertex data and initial partition information (`graph->home`).
        7.  Allocates workspace memory (`AllocateWSpace`).
        8.  Sets partitioning parameters like `ipc_factor` and `CoarsenTo`.
        9.  Calls the core recursive partitioning function `Adaptive_Partition`.
        10. Remaps the graph data based on the new partition (`ParallelReMapGraph`).
        11. Copies the resulting partition to the output `part` array and sets `edgecut`.
        12. Prints timing and post-partitioning info if debug flags are set.
        13. Frees graph structures and control structures.
        14. Reverts numbering if `numflag` was 1.
        15. Checks for memory leaks.
        16. Returns `METIS_OK` on success or `METIS_ERROR` on failure.

*   **`Adaptive_Partition(ctrl_t *ctrl, graph_t *graph)`**:
    *   This is an internal, recursive function that implements the multilevel adaptive partitioning strategy.
    *   **Parameters**:
        *   `ctrl`: The control structure.
        *   `graph`: The current graph structure (could be a coarsened version of the original).
    *   **Functionality (Recursive Multilevel Approach)**:
        1.  Sets up communication structures for the current graph level (`CommSetup`).
        2.  Calculates a `redist_factor` based on global edge weight and vertex size sums, influencing balance between cut and redistribution.
        3.  **Base Case**: If the graph is small enough (compared to `ctrl->CoarsenTo`) or significantly smaller than the next finer graph:
            *   Allocates refinement workspace.
            *   Initializes `graph->where` from `graph->home`.
            *   Balances the partition if it's significantly imbalanced (`Balance_Partition`), especially if not in a pure refinement mode.
            *   If it's the original graph (no finer graph exists), it may perform initial balancing and then adaptive refinement (`KWayBalance`, `KWayAdaptiveRefine`).
        4.  **Recursive Step**: If the graph is large enough:
            *   Coarsens the graph: Selects matching strategy (`Match_Local` for coupled, `Match_Global` for uncoupled) to create `graph->coarser`.
            *   Recursively calls `Adaptive_Partition` on `graph->coarser`.
            *   Projects the partition from `graph->coarser` back to the current `graph` (`ProjectPartition`).
            *   Computes partition parameters for the current level.
            *   If multiple constraints exist and the graph is not too coarse, it may perform balancing (`KWayBalance`) if imbalance is detected.
            *   Performs adaptive k-way refinement (`KWayAdaptiveRefine`) to improve the projected partition.
        5.  Debug progress information is printed if enabled.

## Important Variables/Constants

*   **Input/Output Parameters (for `ParMETIS_V3_AdaptiveRepart`)**:
    *   `vtxdist`, `xadj`, `adjncy`: Define the distributed graph.
    *   `vwgt`, `vsize`, `adjwgt`: Define vertex weights, sizes, and edge weights.
    *   `ncon`, `nparts`: Define number of constraints and target partitions.
    *   `tpwgts`, `ubvec`: Define target partition weights and imbalance tolerance.
    *   `ipc2redist`: Factor balancing cut minimization and data redistribution.
    *   `options`: Control various behaviors.
    *   `edgecut`, `part`: Output variables for cut and partition.
    *   `comm`: MPI communicator.
*   **Core Internal Structures**:
    *   `ctrl_t *ctrl`: Control structure holding global settings, MPI info, options.
    *   `graph_t *graph`: Graph structure holding local graph data, partition info, and pointers to coarser/finer graphs in the multilevel hierarchy.
*   **Key Control Parameters (within `ctrl_t` or derived)**:
    *   `ctrl->ipc_factor`: Importance of inter-processor communication (edge cut).
    *   `ctrl->redist_factor`: Importance of data redistribution/balance.
    *   `ctrl->CoarsenTo`: Target size for the coarsest graph.
    *   `ctrl->ps_relation`: `PARMETIS_PSR_COUPLED` or `PARMETIS_PSR_UNCOUPLED`, affecting matching strategy.
*   **Constants**:
    *   `PARMETIS_OP_AMETIS`: Operation type identifier for adaptive METIS.
    *   `NGR_PASSES`: Default number of passes for certain refinement operations (e.g., `KWayAdaptiveRefine`).
    *   `COARSEN_FRACTION`: Ratio determining when to stop coarsening based on size reduction.
    *   `REFINE_PARTITION`: A mode for `ctrl->partType`, possibly indicating a refinement-only approach.

## Usage Examples

```
N/A for direct usage of Adaptive_Partition.
```
The function `ParMETIS_V3_AdaptiveRepart` is an API function intended to be called by user applications linking against the ParMETIS library. A typical usage would involve:
1.  Setting up the distributed graph data (`vtxdist`, `xadj`, `adjncy`, weights, etc.) on each MPI process.
2.  Defining partitioning parameters (`nparts`, `tpwgts`, `ubvec`, `options`, etc.).
3.  Calling `ParMETIS_V3_AdaptiveRepart` with these parameters.
4.  Using the output `part` array (new partition assignments) and `edgecut` value.

Example call signature:
```c
// Assume all parameters are initialized appropriately
idx_t edgecut;
MPI_Comm comm = MPI_COMM_WORLD; // Or a specific application communicator

int ret = ParMETIS_V3_AdaptiveRepart(vtxdist, xadj, adjncy,
                                     vwgt, vsize, adjwgt,
                                     &wgtflag, &numflag, &ncon, &nparts,
                                     tpwgts, ubvec, &ipc2redist_factor,
                                     options, &edgecut, part, &comm);

if (ret == METIS_OK) {
    // Use the new 'part' array and 'edgecut'
}
```

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Directly included. Provides definitions for `ctrl_t`, `graph_t`, types (`idx_t`, `real_t`), constants, macros, and prototypes for numerous functions used in this file, including:
        *   Input checking: `CheckInputsAdaptiveRepart`
        *   Setup: `SetupCtrl`, `SetupGraph`, `CommSetup`
        *   Memory management: `gk_malloc_init`, `gk_GetCurMemoryUsed`, `gk_malloc_cleanup`, `AllocateWSpace`, `AllocateRefinementWorkSpace`, `rwspacemalloc`, `ismalloc`, `FreeCtrl`, `FreeInitialGraphAndRemap`, `WCOREPUSH`/`POP`
        *   Numbering & Remapping: `ChangeNumbering`, `ParallelReMapGraph`
        *   Core Partitioning Steps: `Match_Local`, `Match_Global` (coarsening), `ProjectPartition`, `Balance_Partition`, `KWayBalance` (balancing), `KWayAdaptiveRefine` (refinement, implemented in `akwayfm.c`).
        *   Parameter computation: `ComputePartitionParams`, `ComputeParallelBalance`
        *   Utilities: `icopy`, `isum`, `ravg`, `GlobalSESum`, `GlobalSEMin`, `GlobalSEMax`, `GlobalSEMinComm`
        *   Debugging & Timing: `IFSET`, `DBG_TIME`, `DBG_PROGRESS`, `DBG_INFO`, `PrintTimingInfo`, `PrintPostPartInfo`, `rprintf`, `STARTTIMER`/`STOPTIMER`
        *   Graph I/O (conditional): `graph_WriteToDisk`, `graph_ReadFromDisk`
    *   `akwayfm.c`: The `KWayAdaptiveRefine` function, a crucial part of the refinement phase, is implemented in this file.
    *   Other `.c` files in `libparmetis/`: Various helper functions for graph manipulation, coarsening strategies, balancing algorithms, communication wrappers, etc., are implemented across different source files.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Essential for parallelism. Used for setting up communicators (`MPI_Comm`), and for various global reduction/communication operations (often via `gkMPI_` wrappers defined in ParMETIS).
*   **Other Interactions**:
    *   The function `ParMETIS_V3_AdaptiveRepart` modifies the user-provided `part` array to store the new partition and `edgecut` to store the resulting cut value.
    *   The behavior of the algorithms is heavily influenced by the `options` array and other numerical parameters (`nparts`, `tpwgts`, `ubvec`, `ipc2redist`).
    *   The multilevel approach involves creating and destroying temporary graph structures (`graph->coarser`, `graph->finer`) during execution.
    *   Memory usage is tracked using `gk_GetCurMemoryUsed` to detect potential leaks.

```
