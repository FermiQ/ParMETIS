# libparmetis/rmetis.c

## Overview

This file is the entry point for the ParMETIS V3 API function `ParMETIS_V3_RefineKway`. This function is designed to refine an existing k-way partition of a distributed graph. It uses a parallel multilevel approach that involves coarsening the graph, refining the partition at the coarsest level (often by leveraging adaptive partitioning techniques), and then projecting the refinement back through the levels to the original graph.

## Key Components

### Functions/Classes/Modules

*   **`ParMETIS_V3_RefineKway(idx_t *vtxdist, idx_t *xadj, idx_t *adjncy, idx_t *vwgt, idx_t *adjwgt, idx_t *wgtflag, idx_t *numflag, idx_t *ncon, idx_t *nparts, real_t *tpwgts, real_t *ubvec, idx_t *options, idx_t *edgecut, idx_t *part, MPI_Comm *comm)`**:
    *   This is the V3 API function for k-way partition refinement.
    *   **Parameters**:
        *   `vtxdist`, `xadj`, `adjncy`: Distributed graph structure.
        *   `vwgt`, `adjwgt`: Vertex and edge weights.
        *   `wgtflag`: Flags for presence of weights.
        *   `numflag`: 0-based or 1-based indexing.
        *   `ncon`: Number of constraints.
        *   `nparts`: Number of partitions (must match the existing partition in `part`).
        *   `tpwgts`: Target partition weights.
        *   `ubvec`: Load imbalance tolerance.
        *   `options`: ParMETIS options array.
        *   `edgecut` (output): Stores the edge cut of the refined partition.
        *   `part` (input/output): Input is the existing partition; output is the refined partition.
        *   `comm`: MPI communicator.
    *   **Functionality**:
        1.  Performs input validation using `CheckInputsPartKway` (the same check as for k-way partitioning from scratch, as many parameters are similar).
        2.  Initializes memory management and sets up the `ctrl_t` control structure with `optype = PARMETIS_OP_RMETIS`.
        3.  Handles the trivial case of `nparts == 1`.
        4.  If `numflag > 0`, converts graph numbering to 0-based (`ChangeNumbering`).
        5.  Sets up the `graph_t` structure (`SetupGraph`).
        6.  Initializes `graph->home`:
            *   If `ctrl->ps_relation == PARMETIS_PSR_COUPLED`, `home` is set to the current PE ID for all local vertices.
            *   Otherwise (uncoupled), `home` is initialized with the input `part` array. This `home` array represents the initial partition that will be refined.
        7.  Allocates workspace (`AllocateWSpace`).
        8.  Sets `ctrl->CoarsenTo`, a threshold for stopping coarsening (typically larger for refinement than for partitioning from scratch, to allow more levels for refinement to work on).
        9.  Calls `Adaptive_Partition(ctrl, graph)`. Despite its name, when `ctrl->partType` is `REFINE_PARTITION` (as set by `PARMETIS_OP_RMETIS`), `Adaptive_Partition` executes refinement-focused logic. This usually involves coarsening the graph while respecting the `home` assignments, potentially performing some balancing or refinement at the coarsest level, and then uncoarsening with refinement steps like `KWayAdaptiveRefine`.
        10. Remaps the graph based on the refined partition (`ParallelReMapGraph`).
        11. Copies the refined partition from `graph->where` to the output `part` array.
        12. Stores the computed edge cut in `*edgecut`.
        13. Prints timing and statistics if debug flags are set.
        14. Frees graph structures and control structures.
        15. If `numflag > 0`, converts numbering back to 1-based.
        16. Checks for memory leaks.

## Important Variables/Constants

*   **`ctrl->partType`**: Set to `REFINE_PARTITION` via `PARMETIS_OP_RMETIS`.
*   **`ctrl->ps_relation`**: `PARMETIS_PSR_COUPLED` or `PARMETIS_PSR_UNCOUPLED`, influences how `graph->home` is set and potentially affects coarsening/refinement strategies.
*   **`graph->home`**: Stores the initial partition to be refined.
*   **`graph->where`**: Stores the refined partition after the process.
*   All other parameters are standard for ParMETIS partitioning/refinement calls.

## Usage Examples

```c
// Conceptual call to ParMETIS_V3_RefineKway
// (assuming vtxdist, xadj, adjncy, part (existing partition), comm, etc. are set up)
// idx_t options[PARMETIS_MAX_OPTIONS]; options[0] = 0; // Default options
// idx_t edgecut_val;
//
// // part contains the initial k-way partition to be refined
// ParMETIS_V3_RefineKway(vtxdist, xadj, adjncy, vwgt, adjwgt,
//                        &wgtflag, &numflag, &ncon, &nparts_val,
//                        tpwgts, ubvec, options,
//                        &edgecut_val, part, &comm);
//
// // 'part' is now updated with the refined partition.
// // 'edgecut_val' is the new edge cut.
```
This is an API function.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Core definitions.
    *   `ctrl.c`: For `SetupCtrl`, `FreeCtrl`.
    *   `graph.c`: For `SetupGraph`, `FreeInitialGraphAndRemap`.
    *   `wspace.c`: For `AllocateWSpace`.
    *   `comm.c`: For `CheckInputsPartKway`, `GlobalSEMinComm`.
    *   `ametis.c`: `Adaptive_Partition` is the core multilevel routine called.
    *   `remap.c`: For `ParallelReMapGraph`.
    *   `util.c`: For `ChangeNumbering`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: For all parallel communication.
*   **Other Interactions**:
    *   This function takes an existing partition as input (`part` array, copied to `graph->home`) and aims to improve its quality (reduce edge cut) while maintaining balance.
    *   The behavior of the underlying `Adaptive_Partition` function changes based on `ctrl->partType` being `REFINE_PARTITION`.

```
