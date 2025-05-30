# programs/ptest.c

## Overview

This file is a test driver program for ParMETIS, named `ptest`. It's very similar in structure and purpose to `otest.c`. It's designed to test a comprehensive suite of ParMETIS V3 API functions, including k-way partitioning (`ParMETIS_V3_PartKway`), geometric k-way partitioning (`ParMETIS_V3_PartGeomKway`), geometric partitioning based on coordinates (`ParMETIS_V3_PartGeom`), k-way partition refinement (`ParMETIS_V3_RefineKway`), and adaptive repartitioning (`ParMETIS_V3_AdaptiveRepart`). It does *not* explicitly test the node nested dissection ordering functions (`ParMETIS_V3_NodeND`) like `otest.c` does.

## Key Components

### Functions/Classes/Modules

*   **`main(int argc, char *argv[])`**:
    *   Entry point for the `ptest` program.
    *   Initializes MPI.
    *   Requires one or two command-line arguments:
        1.  `graph-file`: The name of the graph file.
        2.  `coord-file` (optional): The name of the vertex coordinates file.
    *   Calls `TestParMetis_GPart` to run the actual tests.
    *   Finalizes MPI.
*   **`TestParMetis_GPart(char *filename, char *xyzfile, MPI_Comm comm)`**:
    *   The main testing logic. (Note: This function has the same name as the one in `otest.c` but the content of `otest.c`'s `TestParMetis` is more comprehensive. This version in `ptest.c` might be an older or slightly different variant).
    *   Reads the graph using `ParallelReadGraph`.
    *   If `xyzfile` is provided, reads coordinates using `ReadTestCoordinates`.
    *   Initializes `vwgt` with dummy weights (all 1s, or adapted weights for multi-constraint tests).
    *   **Tests `ParMETIS_V3_PartKway`**:
        *   Iterates through different numbers of partitions (`nparts` from `2*npes` down to `npes/2`).
        *   Iterates through number of constraints (`ncon` from 1 up to `NCON` (defined as 5)).
        *   If `ncon > 1`, calls `Mc_AdaptGraph` to create varying vertex weights.
        *   Calls `ParMETIS_V3_PartKway`.
        *   Computes the actual cut using `ComputeRealCut` and compares it with the cut reported by ParMETIS.
    *   **Tests `ParMETIS_V3_RefineKway` (Uncoupled)**:
        *   After each `PartKway` test, it immediately tests `RefineKway` on that result with `options[3] = PARMETIS_PSR_UNCOUPLED`.
    *   **Tests `ParMETIS_V3_PartGeomKway`**:
        *   If `xyzfile` was provided, runs similar loops for `nparts` and `ncon` as for `PartKway`.
        *   Calls `ParMETIS_V3_PartGeomKway`.
        *   Compares reported cut with `ComputeRealCut`.
    *   **Tests `ParMETIS_V3_PartGeom`**:
        *   If `xyzfile` was provided.
        *   Calls `ParMETIS_V3_PartGeom`.
        *   Computes and prints the cut using `ComputeRealCut`.
    *   **Tests `ParMETIS_V3_RefineKway` (Coupled, after graph move)**:
        1.  Partitions the graph using `PartKway` (quietly) to get an `npes`-way partition.
        2.  Moves the graph using `TestMoveGraph` to create `mgraph`.
        3.  Iterates `ncon` from 1 to `NCON`.
        4.  Calls `ParMETIS_V3_RefineKway` on `mgraph` with `options[3] = PARMETIS_PSR_COUPLED`.
        5.  Computes cut using `ComputeRealCutFromMoved` and compares.
    *   **Tests `ParMETIS_V3_AdaptiveRepart`**:
        1.  Uses `mgraph`. Adapts its vertex weights using `AdaptGraph` (for `ncon=1`) or `Mc_AdaptGraph` (for `ncon>1`).
        2.  First, partitions `mgraph` with `PartKway` to get a base partition (`savepart`).
        3.  Iterates `nparts`, `ncon`, and `ipc2redist` values.
        4.  Calls `ParMETIS_V3_AdaptiveRepart` using `savepart` as the initial state for `mpart`.
        5.  Computes cut using `ComputeRealCutFromMoved` and compares.
*   **`ComputeRealCut(idx_t *vtxdist, idx_t *part, char *filename, MPI_Comm comm)`**:
    *   (Identical to the one in `otest.c`) Gathers the distributed partition vector `part` onto PE 0. PE 0 reads the original graph (serially using `ReadMetisGraph`) and computes the actual edge cut.
*   **`ComputeRealCutFromMoved(idx_t *vtxdist, idx_t *mvtxdist, idx_t *part, idx_t *mpart, char *filename, MPI_Comm comm)`**:
    *   (Identical to `ComputeRealCut2` in `otest.c`) Computes the cut of an original graph based on a partition (`mpart`) of a moved graph. It reconstructs the permutation from the original `part` (which led to the move) to map vertices correctly.
*   **`TestMoveGraph(graph_t *ograph, graph_t *omgraph, idx_t *part, MPI_Comm comm)`**:
    *   (Identical to the one in `otest.c`) A wrapper to test `MoveGraph` from `libparmetis`.
*   **`TestSetUpGraph(ctrl_t *ctrl, idx_t *vtxdist, ..., idx_t wgtflag)`**:
    *   (Identical to `SetUpGraph` in `otest.c`) An older/internal style graph setup wrapper.

## Important Variables/Constants

*   `NCON`: Defined as 5, the maximum number of constraints tested in loops.
*   `graph` (graph_t): Holds the primary distributed graph.
*   `mgraph` (graph_t): Holds the "moved" graph after redistribution.
*   `part`, `mpart`, `savepart`: Arrays for storing partitioning results.
*   `xyz`: Array for vertex coordinates.
*   `options`: Array to control ParMETIS behavior.
*   `MAXNCON`: Used for allocating `tpwgts` and `ubvec`.

## Usage Examples

```bash
# Command-line execution:
mpirun -np <num_procs> ptest <graph_file_name> [coord_file_name]

# Example:
mpirun -np 4 ptest mygraph.graph mygraph.xyz
# This would partition mygraph.graph (using its coordinates from mygraph.xyz where applicable)
# using various ParMETIS routines and parameters.
```

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetisbin.h`: Includes `parmetislib.h` and `programs/proto.h`.
    *   `programs/io.c`: For `ParallelReadGraph`, `ReadTestCoordinates`, `ReadMetisGraph`.
    *   `programs/adaptgraph.c`: For `AdaptGraph`, `Mc_AdaptGraph`.
    *   `libparmetis/parmetis.h` (API): Calls `ParMETIS_V3_PartKway`, `ParMETIS_V3_PartGeomKway`, `ParMETIS_V3_PartGeom`, `ParMETIS_V3_RefineKway`, `ParMETIS_V3_AdaptiveRepart`.
    *   `libparmetis` (internal functions): Uses `SetupCtrl`, `MoveGraph`, `FreeGraph`, etc. (Some via local wrappers like `TestMoveGraph` and `TestSetUpGraph`).
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`.
    *   Standard C library.
*   **Other Interactions**:
    *   `ptest.c` is a comprehensive test driver focusing on partitioning and refinement APIs. Unlike `otest.c`, it does not explicitly include tests for `ParMETIS_V3_NodeND`.
    *   It systematically varies parameters like `nparts`, `ncon`, and some `options` to cover a range of scenarios.
    *   The `ComputeRealCut` functions are important for verifying the edge cut reported by ParMETIS against a value computed from the full, gathered graph structure.

```
