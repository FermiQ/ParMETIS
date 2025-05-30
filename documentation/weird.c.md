# libparmetis/weird.c

## Overview

This file contains input validation functions for various ParMETIS V3 API routines. These functions check for `NULL` pointers for mandatory arguments and verify that critical parameters (like `ncon`, `nparts`, `ndims`, `ubvec` values, `tpwgts` sums) are within valid and reasonable ranges. If an error is detected, they print an error message and return 0 (failure), otherwise 1 (success). It also contains `PartitionSmallGraph` which is a utility for handling graphs deemed too small for parallel processing.

## Key Components

### Functions/Classes/Modules

*   **Input Checking Macros**:
    *   `RIFN(x)`: "Return If Null". Checks if pointer `x` is `NULL`. If so, prints an error and returns 0.
    *   `RIFNP(x)`: "Return If Not Positive". Checks if the value pointed to by `x` is `<= 0`. If so, prints an error and returns 0.

*   **Input Validation Functions**:
    *   **`CheckInputsPartKway(...)`**: Validates inputs for `ParMETIS_V3_PartKway` and `ParMETIS_V3_RefineKway`.
        *   Checks `vtxdist`, `xadj`, `adjncy`, `wgtflag`, `numflag`, `ncon`, `nparts`, `tpwgts`, `ubvec`, `options`, `edgecut`, `part`.
        *   If weights are indicated by `wgtflag`, checks `vwgt` and `adjwgt`.
        *   Ensures sum of `vwgt` for any constraint is not zero.
        *   Ensures local `nvtxs > 0`.
        *   Ensures `*ncon > 0`, `*nparts > 0`.
        *   Validates `tpwgts` sum to 1.0 per constraint and values are in [0, 1.001].
        *   Validates `ubvec` values are `> 1.0`.
    *   **`CheckInputsPartGeomKway(...)`**: Validates inputs for `ParMETIS_V3_PartGeomKway`.
        *   Similar checks as `CheckInputsPartKway`.
        *   Additionally checks `xyz`, `ndims`.
        *   Ensures `*ndims > 0` and `*ndims <= 3`.
    *   **`CheckInputsPartGeom(...)`**: Validates inputs for `ParMETIS_V3_PartGeom`.
        *   Checks `vtxdist`, `xyz`, `ndims`, `part`.
        *   Ensures local `nvtxs > 0`.
        *   Ensures `*ndims > 0` and `*ndims <= 3`.
    *   **`CheckInputsAdaptiveRepart(...)`**: Validates inputs for `ParMETIS_V3_AdaptiveRepart`.
        *   Similar checks as `CheckInputsPartKway`.
        *   Additionally checks `ipc2redist` to be within a valid range [0.0001, 1000000.0].
        *   `vsize` is not explicitly checked for NULL with `RIFN` but is implicitly used if weights are present.
    *   **`CheckInputsNodeND(...)`**: Validates inputs for `ParMETIS_V3_NodeND`.
        *   Checks `vtxdist`, `xadj`, `adjncy`, `numflag`, `options`, `order`, `sizes`.
        *   Ensures local `nvtxs > 0`.
    *   **`CheckInputsPartMeshKway(...)`**: Validates inputs for `ParMETIS_V3_PartMeshKway`.
        *   Checks `elmdist`, `eptr`, `eind`, `wgtflag`, `numflag`, `ncon`, `nparts`, `tpwgts`, `ubvec`, `options`, `edgecut`, `part`.
        *   If element weights are indicated, checks `elmwgt`.
        *   Ensures local `nelms > 0`.
        *   Ensures `*ncon > 0`, `*nparts > 0`.
        *   Validates `tpwgts` and `ubvec` like other functions.

*   **Utility Function**:
    *   **`PartitionSmallGraph(ctrl_t *ctrl, graph_t *graph)`**:
        *   Handles partitioning of graphs deemed too small for parallel processing.
        *   It assembles the entire graph onto PE 0 using `AssembleAdaptiveGraph`.
        *   PE 0 then calls serial METIS (`METIS_PartGraphKway`) to partition the assembled graph.
        *   The best partition among those computed by PEs (if multiple PEs ran this, though typically it implies full assembly on PE0 which then broadcasts/scatters) is selected using `MPI_MINLOC` on the cut size.
        *   The resulting partition is scattered back to all processors.
        *   Finally, global partition weights (`gnpwgts`) are computed.

## Important Variables/Constants

The functions primarily operate on the parameters passed to them, which correspond to the ParMETIS API arguments.

## Usage Examples

```
N/A
```
These are internal ParMETIS functions.
*   The `CheckInputs*` functions are called at the beginning of their respective ParMETIS API routines to ensure the user has provided valid parameters. If validation fails, the API function typically returns `METIS_ERROR` after `GlobalSEMinComm` confirms an error on at least one PE.
*   `PartitionSmallGraph` is called by routines like `ParMETIS_V3_PartKway` if the graph size is below a certain threshold (e.g., `SMALLGRAPH`).

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides `ctrl_t`, `graph_t` definitions, `idx_t`, `real_t` types, MPI wrappers (`gkMPI_Comm_rank`, `GlobalSESumComm`), and GKlib utilities (`isum`, `rsum`).
    *   `initbalance.c`: `AssembleAdaptiveGraph` is used by `PartitionSmallGraph`.
    *   `graph.c`: `FreeGraph` is used by `PartitionSmallGraph`.
    *   `metis.h` (external METIS library): `METIS_SetDefaultOptions`, `METIS_PartGraphKway` are used by `PartitionSmallGraph`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Used for communication in `PartitionSmallGraph` and by `GlobalSEMinComm`/`GlobalSESumComm` in the check functions.
    *   `METIS`: Serial METIS is used by `PartitionSmallGraph`.
    *   Standard C library: `stdio.h` (for `printf`), `stdlib.h` (for `abort`).
*   **Other Interactions**:
    *   The input check functions are the first line of defense against user error for API calls.
    *   `PartitionSmallGraph` provides a fallback to serial METIS for small graphs, avoiding the overhead and complexity of parallel algorithms where they might not be beneficial.

```
