# libparmetis/mmetis.c

## Overview

This file serves as the entry point for `ParMETIS_V3_PartMeshKway`, a ParMETIS API function designed for partitioning a distributed mesh into `k` parts. It achieves this by first converting the mesh into its dual graph representation and then partitioning this dual graph using the standard parallel k-way graph partitioning algorithm (`ParMETIS_V3_PartKway`).

## Key Components

### Functions/Classes/Modules

*   **`ParMETIS_V3_PartMeshKway(idx_t *elmdist, idx_t *eptr, idx_t *eind, idx_t *elmwgt, idx_t *wgtflag, idx_t *numflag, idx_t *ncon, idx_t *ncommon, idx_t *nparts, real_t *tpwgts, real_t *ubvec, idx_t *options, idx_t *edgecut, idx_t *part, MPI_Comm *comm)`**:
    *   This is the V3 API function for k-way partitioning of a distributed mesh.
    *   **Parameters**:
        *   `elmdist`: Distribution of mesh elements among processors.
        *   `eptr`, `eind`: CSR-like representation of the mesh (elements to nodes).
        *   `elmwgt`: Weights of the mesh elements (if `wgtflag` indicates).
        *   `wgtflag`: Flags for presence of element weights.
        *   `numflag`: 0-based or 1-based indexing for input `eind`.
        *   `ncon`: Number of constraints for element weights.
        *   `ncommon`: Number of common nodes defining a shared face (for dual graph construction).
        *   `nparts`: Desired number of partitions.
        *   `tpwgts`: Target partition weights.
        *   `ubvec`: Load imbalance tolerance.
        *   `options`: ParMETIS options array.
        *   `edgecut` (output): Stores the edge cut of the partition on the dual graph.
        *   `part` (output): Stores the partition assignment for each local element.
        *   `comm`: MPI communicator.
    *   **Functionality**:
        1.  Performs input validation (`CheckInputsPartMeshKway`).
        2.  Initializes memory management and sets up a minimal `ctrl_t` structure (primarily for communicator and debug level).
        3.  **Mesh to Dual Conversion**: Calls `ParMETIS_V3_Mesh2Dual` to convert the input mesh into its dual graph. The dual graph's adjacency structure is returned in `xadj` and `adjncy`. The vertices of the dual graph correspond to the elements of the original mesh, so `elmdist` serves as the `vtxdist` for the dual graph, and `elmwgt` serves as its `vwgt`.
        4.  **Dual Graph Partitioning**: Calls `ParMETIS_V3_PartKway` to partition the newly created dual graph. The parameters `elmdist` (as `vtxdist`), `xadj`, `adjncy`, `elmwgt` (as `vwgt`), `wgtflag`, `numflag`, `ncon`, `nparts`, `tpwgts`, `ubvec`, `options` are passed to this function. The resulting partition for the dual graph vertices (which are the mesh elements) is stored in `part`, and the edge cut is stored in `edgecut`.
        5.  Frees the dynamically allocated `xadj` and `adjncy` for the dual graph using `METIS_Free`.
        6.  Frees the control structure and performs memory cleanup.

## Important Variables/Constants

*   **`elmdist`, `eptr`, `eind`, `elmwgt`**: Input mesh data.
*   **`ncommon`**: Parameter for `ParMETIS_V3_Mesh2Dual`.
*   **`xadj`, `adjncy`**: Temporary pointers to store the CSR structure of the dual graph.
*   **`part`, `edgecut`**: Output parameters.

## Usage Examples

```c
// Conceptual call to ParMETIS_V3_PartMeshKway
// (assuming elmdist, eptr, eind, elmwgt, part, comm, etc. are set up)
// idx_t wgtflag = 1; // Assume element weights are provided
// idx_t numflag = 0;
// idx_t ncon = 1;
// idx_t ncommonnodes = 3; // e.g., for 3D tet mesh
// idx_t nparts_val = 4;
// real_t ubvec_val[1]; ubvec_val[0] = 1.05;
// idx_t options_val[PARMETIS_MAX_OPTIONS]; options_val[0] = 0;
// idx_t edgecut_val;
//
// ParMETIS_V3_PartMeshKway(elmdist, eptr, eind, elmwgt,
//                          &wgtflag, &numflag, &ncon, &ncommonnodes, &nparts_val,
//                          NULL, ubvec_val, options_val,
//                          &edgecut_val, part, &comm);
//
// // 'part' now contains the partition for each local element.
// // 'edgecut_val' is the cut on the dual graph.
```
This is an API function.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Core definitions.
    *   `ctrl.c`: For `SetupCtrl`, `FreeCtrl`.
    *   `comm.c`: For `CheckInputsPartMeshKway`, `GlobalSEMinComm`, `GlobalSESum`.
    *   `mesh.c`: `ParMETIS_V3_Mesh2Dual` is called to perform the mesh-to-dual conversion.
    *   `kmetis.c`: `ParMETIS_V3_PartKway` is called to partition the generated dual graph.
    *   `metis.h` (external METIS library): `METIS_Free` is used to deallocate the dual graph structure.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Used by underlying functions.
*   **Other Interactions**:
    *   This function provides a convenient way to partition a mesh by leveraging graph partitioning algorithms on its dual. The quality of the mesh partition depends on how well the dual graph representation captures the connectivity and balance requirements of the original mesh.
    *   The `elmwgt` (element weights) become the `vwgt` (vertex weights) for the dual graph partitioning.

```
