# libparmetis/gkmetis.c

## Overview

This file serves as an entry point for ParMETIS routines that perform graph partitioning using geometric information (vertex coordinates). It provides functions that can partition a graph based solely on coordinates or use coordinates to establish an initial distribution for a k-way partitioning algorithm. The key idea is to leverage spatial proximity to guide the partitioning process, which can be particularly effective for graphs derived from geometric meshes.

## Key Components

### Functions/Classes/Modules

*   **`ParMETIS_V3_PartGeomKway(idx_t *vtxdist, idx_t *xadj, ..., MPI_Comm *comm)`**:
    *   This is a V3 API function that partitions a distributed graph into `nparts` partitions.
    *   It first uses geometric partitioning (based on `xyz` coordinates and `ndims`) to achieve an initial `npes`-way partition (distributing the graph among processors).
    *   Then, it moves the graph data according to this initial partition (`MoveGraph`).
    *   Finally, it partitions the (now potentially redistributed) local graph on each processor into `nparts/npes` (or similar) local partitions using a parallel k-way partitioning algorithm (`Global_Partition` or `PartitionSmallGraph` for small graphs).
    *   The final partition is then projected back to the original graph distribution.
    *   Handles the `npes == 1` case by calling the serial METIS `METIS_PartGraphKway`.
    *   Handles the `nparts == 1` trivial case.
*   **`ParMETIS_V3_PartGeom(idx_t *vtxdist, idx_t *ndims, real_t *xyz, idx_t *part, MPI_Comm *comm)`**:
    *   This is a V3 API function that partitions the vertices based *solely* on their geometric coordinates (`xyz` and `ndims`) into `npes` partitions (one partition per processor).
    *   It creates a "fake" graph structure because the underlying partitioning machinery (`Coordinate_Partition`) expects a graph.
    *   The resulting partition assigns each vertex to one of the processors.
    *   Handles the `npes == 1` trivial case.

## Important Variables/Constants

*   **`xyz`**: Array of real numbers storing the coordinates of the vertices. For `N` vertices and `D` dimensions, this would be an array of size `N*D`.
*   **`ndims`**: The number of dimensions for the coordinates in `xyz`.
*   **`vtxdist`**: Standard ParMETIS array defining the distribution of vertices across processors.
*   **`xadj`, `adjncy`, `vwgt`, `adjwgt`**: Standard graph CSR representation and weights.
*   **`part`**: Output array where the partition assignments are stored.
*   **`nparts`**: The desired number of partitions for `ParMETIS_V3_PartGeomKway`.
*   **`ctrl_t`**: The ParMETIS control structure.
*   **`graph_t`**: The ParMETIS graph structure.
*   **`mgraph`**: A "moved graph" structure used in `ParMETIS_V3_PartGeomKway` after the initial geometric distribution.
*   **`METIS_NOPTIONS`, `METIS_OPTION_NUMBERING`**: Used when calling serial METIS for the `npes == 1` case.
*   `SMALLGRAPH`: A threshold to decide whether to partition serially or in parallel after the initial geometric distribution.

## Usage Examples

```c
// Conceptual call to ParMETIS_V3_PartGeomKway
// (assuming all parameters like vtxdist, xadj, xyz, part, comm are set up)
// idx_t options[PARMETIS_MAX_OPTIONS];
// options[0] = 0; // Use default options
// idx_t edgecut;
// ParMETIS_V3_PartGeomKway(vtxdist, xadj, adjncy, vwgt, adjwgt,
//                          &wgtflag, &numflag, &ndims, xyz,
//                          &ncon, &nparts, tpwgts, ubvec,
//                          options, &edgecut, part, &comm);

// Conceptual call to ParMETIS_V3_PartGeom
// idx_t part[nvtxs_local];
// ParMETIS_V3_PartGeom(vtxdist, &ndims, xyz_local, part, &comm);
// // part now contains the PE assignment for each local vertex.
```
These are API functions intended to be called by user applications.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides core data structures (`ctrl_t`, `graph_t`), type definitions, and prototypes for many internal functions.
    *   `ctrl.c`: For `SetupCtrl`, `FreeCtrl`.
    *   `graph.c`: For `SetupGraph`, `MoveGraph`, `FreeGraph`, `FreeInitialGraphAndRemap`, `SetupGraph_nvwgts`.
    *   `wspace.c`: For `AllocateWSpace`.
    *   `xyzpart.c`: For `Coordinate_Partition` (the core geometric partitioning routine).
    *   `kmetis.c` (indirectly): For `Global_Partition` (parallel k-way partitioning).
    *   `serial.c`: For `PartitionSmallGraph`.
    *   `remap.c`: For `ParallelReMapGraph`, `ProjectInfoBack`.
    *   `util.c`: For `ChangeNumbering`.
    *   `comm.c`: For `GlobalSEMinComm`, `GlobalSESum`, `GlobalSEMax`, `CommInterfaceData`.
    *   `metis.h` (external METIS library): For `METIS_PartGraphKway`, `METIS_SetDefaultOptions` when `npes == 1`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: For all parallel communication.
    *   `METIS`: The serial METIS library is called as a fallback if `npes == 1`.
*   **Other Interactions**:
    *   These functions are often used as a first step in partitioning physical meshes, where vertex coordinates provide a good heuristic for initial partitioning.
    *   The quality of the geometric partition can influence the final result of `ParMETIS_V3_PartGeomKway`.

```
