# programs/otest.c

## Overview

This file serves as a test driver program for various ParMETIS V3 API functions, including k-way partitioning (`ParMETIS_V3_PartKway`), geometric k-way partitioning (`ParMETIS_V3_PartGeomKway`), geometric partitioning based on coordinates only (`ParMETIS_V3_PartGeom`), k-way partition refinement (`ParMETIS_V3_RefineKway`), adaptive repartitioning (`ParMETIS_V3_AdaptiveRepart`), and node nested dissection ordering (`ParMETIS_V3_NodeND`). It reads a graph and optionally its coordinates, then calls these API functions with a range of parameters (number of constraints, number of parts, different internal options) to test their functionality and reports the resulting edge cuts.

## Key Components

### Functions/Classes/Modules

*   **`main(idx_t argc, char *argv[])`**:
    *   Entry point for the `otest` program.
    *   Initializes MPI.
    *   Requires one command-line argument: the graph filename.
    *   Calls `TestParMetis_GPart` (which seems to be a typo and should be `TestParMetis`, as defined in this file) to run the tests.
    *   Finalizes MPI.
*   **`TestParMetis(char *filename, MPI_Comm comm)`**:
    *   The main testing logic.
    *   Reads the graph using `ParallelReadGraph` and coordinates using `ReadTestCoordinates`.
    *   Iterates through different numbers of partitions (`nparts`) and constraints (`ncon`).
    *   For multi-constraint tests, it calls `Mc_AdaptGraph` to create varying vertex weights based on a temporary partition. For single constraint, it sets unit vertex weights.
    *   **Tests `ParMETIS_V3_PartKway`**: Calls with `ncon` and `nparts`, and with `options[2]` (related to internal strategy) set to 1 and 2.
    *   **Tests `ParMETIS_V3_PartGeomKway`**: Calls with similar parameters as `PartKway`, including coordinates.
    *   **Tests `ParMETIS_V3_PartGeom`**: (Currently commented out in the provided code).
    *   **Tests `ParMETIS_V3_RefineKway`**:
        *   First, calls it on the initial graph state.
        *   Then, computes a partition using `ParMETIS_V3_PartKway` (quietly, `options[0]=0`), moves the graph using `TestMoveGraph` to create `mgraph`, and then calls `RefineKway` on `mgraph` for various `ncon` and `options[2]`.
    *   **Tests `ParMETIS_V3_AdaptiveRepart`**:
        *   Uses the `mgraph` (moved graph from previous step).
        *   Modifies vertex weights of `mgraph` using `AdaptGraph` (for `ncon=1`) or `Mc_AdaptGraph` (for `ncon>1`).
        *   Calls `AdaptiveRepart` with varying `ipc2redist` values and `options[2]`.
    *   **Tests `ParMETIS_V3_NodeND`**: Calls with `options[PMV3_OPTION_IPART]` (initial partitioning type for ND) set to 1 and 2.
    *   Prints reported edge cuts for each test.
*   **`ComputeRealCut(idx_t *vtxdist, idx_t *part, char *filename, MPI_Comm comm)`**:
    *   Gathers the distributed partition vector `part` onto PE 0.
    *   PE 0 reads the original graph (serially using `ReadMetisGraph`) and computes the actual edge cut based on `gpart`.
*   **`ComputeRealCut2(idx_t *vtxdist, idx_t *mvtxdist, idx_t *part, idx_t *mpart, char *filename, MPI_Comm comm)`**:
    *   Similar to `ComputeRealCut`, but for a scenario involving an original partition `part` and a partition `mpart` on a moved graph.
    *   It reconstructs the permutation (`perm`) to map original vertex indices to moved graph vertex indices based on `gpart` (gathered `part`).
    *   Then computes the cut on the original graph structure using `gmpart[perm[vertex]]`.
*   **`TestMoveGraph(graph_t *ograph, graph_t *omgraph, idx_t *part, MPI_Comm comm)`**:
    *   A wrapper to test the `MoveGraph` functionality from `libparmetis`.
    *   It sets up a temporary `ctrl_t` and `wspace_t`, calls `Mc_SetUpGraph` (seems to be an older or internal version of `SetupGraph`), then `SetUp` (unknown, possibly older comm setup), then `MoveGraph`.
    *   Copies key fields from the returned `mgraph` to `omgraph`.
*   **`SetUpGraph(ctrl_t *ctrl, idx_t *vtxdist, ..., idx_t wgtflag)`**:
    *   A wrapper around `Mc_SetUpGraph` (older/internal graph setup) for use within `TestMoveGraph`.

## Important Variables/Constants

*   `graph` (graph_t): Holds the primary distributed graph.
*   `mgraph` (graph_t): Holds the "moved" graph after redistribution.
*   `part`, `mpart`, `savepart`, `order`, `sizes`: Arrays for storing partitioning and ordering results.
*   `xyz`: Array for vertex coordinates.
*   `options`: Array to control ParMETIS behavior.
*   `MAXNCON`: Used for allocating `tpwgts` and `ubvec`.

## Usage Examples

```bash
# Command-line execution:
mpirun -np <num_procs> otest <graph_file_name>

# Example:
mpirun -np 4 otest mygraph.graph
```
This program reads `mygraph.graph` and its coordinate file (`mygraph.graph.xyz`), then runs a battery of tests on it using 4 processors.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetisbin.h`: Includes `parmetislib.h` and `programs/proto.h`.
    *   `programs/io.c`: For `ParallelReadGraph`, `ReadTestCoordinates`, `ReadMetisGraph`.
    *   `programs/adaptgraph.c`: For `AdaptGraph`, `Mc_AdaptGraph`.
    *   `libparmetis/parmetis.h` (API): Calls `ParMETIS_V3_PartKway`, `ParMETIS_V3_PartGeomKway`, `ParMETIS_V3_RefineKway`, `ParMETIS_V3_AdaptiveRepart`, `ParMETIS_V3_NodeND`.
    *   `libparmetis` (internal functions, likely through `parmetisbin.h`): `SetUpCtrl`, `Mc_SetUpGraph` (older graph setup), `MoveGraph`, `FreeInitialGraphAndRemap`, `FreeGraph`, `FreeWSpace`, `MALLOC_CHECK`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`.
    *   Standard C library.
*   **Other Interactions**:
    *   `otest.c` is a comprehensive test driver that exercises multiple ParMETIS API functions with varied parameters, making it useful for regression testing and validation.
    *   The `TestMoveGraph` and the local `SetUpGraph` wrapper suggest it might be testing against an older or specific internal way of setting up and moving graphs, possibly for backward compatibility checks or focused testing of those components.
    *   The file `ptest.c` is mentioned in the original `$Id$` comment, and `otest.c` seems to be a modified or evolved version of it, likely focusing more on ordering (`_NodeND`) tests.

```
