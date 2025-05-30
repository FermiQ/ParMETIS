# libparmetis/mesh.c

## Overview

This file contains the implementation of `ParMETIS_V3_Mesh2Dual`, a ParMETIS API function. Its purpose is to convert a distributed mesh (defined by elements and their constituent nodes) into its dual graph. In the dual graph, each mesh element becomes a vertex, and an edge connects two dual graph vertices if their corresponding mesh elements share a common face (defined by `ncommon` common nodes).

## Key Components

### Functions/Classes/Modules

*   **`ParMETIS_V3_Mesh2Dual(idx_t *elmdist, idx_t *eptr, idx_t *eind, idx_t *numflag, idx_t *ncommon, idx_t **r_xadj, idx_t **r_adjncy, MPI_Comm *comm)`**:
    *   This is the V3 API function for mesh-to-dual conversion.
    *   **Parameters**:
        *   `elmdist`: Distribution of elements among processors.
        *   `eptr`: CSR-like pointer array for `eind`, indicating nodes per element.
        *   `eind`: Array storing node IDs for each element.
        *   `numflag`: Indicates 0-based or 1-based numbering for input `eind`.
        *   `ncommon`: The number of common nodes required to define a shared face (e.g., 2 for 2D triangles/quads, 3 for 3D tetrahedra, 4 for 3D hexahedra if faces are quads).
        *   `r_xadj` (output): Pointer to store the `xadj` (adjacency pointers) of the resulting dual graph.
        *   `r_adjncy` (output): Pointer to store the `adjncy` (adjacency list) of the resulting dual graph.
        *   `comm`: MPI communicator.
    *   **Functionality**:
        1.  Handles `numflag` for initial node renumbering if input is 1-based (`ChangeNumberingMesh`).
        2.  Normalizes global node IDs by subtracting the global minimum node ID.
        3.  Constructs a global node distribution (`nodedist`).
        4.  **Local Renumbering & Node-to-Element List Construction (Pass 1 - Local)**:
            *   Each PE creates a local mapping of its unique nodes (`nmap`).
            *   `eind` is updated with these local node IDs.
            *   A `nodelist` (array of `ikv_t`) is created where `key` is global node ID and `val` is the local element ID using that node.
        5.  **Global Node-to-Element List (Pass 2 - Communication)**:
            *   `nodelist` entries are sent to PEs that own the respective global nodes (`gkMPI_Alltoallv`).
            *   Each PE constructs `gnptr`, `gnind`: a list of elements for each of its assigned global nodes.
        6.  **Element Adjacency Information Exchange (Pass 3 - Communication)**:
            *   Each PE determines which other PEs need its `gnind` lists (based on which PEs have elements using its nodes).
            *   The `gnind` lists (elements incident to a node) are sent to those PEs (`gkMPI_Alltoallv`).
            *   Each PE receives lists of elements for nodes that are part of its local elements. This forms `nptr`, `nind` (local view of node-to-global-element list for relevant nodes).
        7.  **Dual Graph Construction (Pass 4 & 5 - Local)**:
            *   Iterates through local elements. For each element `i`:
                *   For each node in element `i`:
                    *   Iterates through other elements `kk` sharing that node (from `nptr`, `nind`).
                    *   A hash table (`htable`) is used to count common nodes between element `i` and element `kk`.
                *   If the count of common nodes for an adjacent element `kk` is `>= *ncommon`, an edge is added in the dual graph between element `i` and element `kk`.
            *   This is done in two passes: first to count degrees (for `myxadj`), then to fill `myadjncy`.
        8.  The output `*r_xadj` and `*r_adjncy` are populated with the CSR structure of the dual graph.
        9.  Node numbering in `eind` is restored to original global values + `gminnode`.
        10. If `numflag` was 1, dual graph numbering is also adjusted back.

## Important Variables/Constants

*   **`elmdist`**: Distribution of elements.
*   **`eptr`, `eind`**: Input mesh connectivity (elements to nodes).
*   **`ncommon`**: Number of common nodes to define an edge in the dual graph.
*   **`nodedist`**: Calculated distribution of global nodes among PEs.
*   **`nodelist (ikv_t *)`**: Temporary list mapping global node IDs to local element IDs.
*   **`gnptr`, `gnind`**: CSR-like list on each PE: for each of its nodes, lists all global elements incident to it.
*   **`nptr`, `nind`**: CSR-like list on each PE: for each local unique node in its elements, lists all global elements incident to it (received from other PEs).
*   **`htable`**: Hash table used during dual graph construction to count shared nodes between elements.
*   **`myxadj`, `myadjncy`**: Output CSR arrays for the dual graph.
*   `mask`: Used for hashing, `(1<<11)-1`.

## Usage Examples

```c
// Conceptual call to ParMETIS_V3_Mesh2Dual
// (assuming elmdist, eptr, eind, comm are set up)
// idx_t numflag = 0;
// idx_t ncommonnodes = 3; // For 3D tetrahedra, for example
// idx_t *xadj_dual, *adjncy_dual;
//
// ParMETIS_V3_Mesh2Dual(elmdist, eptr, eind, &numflag, &ncommonnodes,
//                       &xadj_dual, &adjncy_dual, &comm);
//
// // xadj_dual and adjncy_dual now represent the dual graph
// // where elmdist can be used as its vtxdist.
// // Need to free xadj_dual and adjncy_dual later using METIS_Free.
```
This is an API function.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides `idx_t`, `ikv_t`, MPI wrappers, memory utilities (`ismalloc`, `ikvmalloc`, `imalloc`, `irealloc`, `gk_free`, `gk_malloc_init`, `gk_GetCurMemoryUsed`, `gk_malloc_cleanup`), sorting (`ikvsorti`), math ops (`imin`, `imax`).
    *   `util.c`: `ChangeNumberingMesh`.
    *   `comm.c`: `GlobalSEMinComm`, `GlobalSEMaxComm`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Used extensively for communication (`gkMPI_Comm_size`, `gkMPI_Comm_rank`, `gkMPI_Alltoall`, `gkMPI_Alltoallv`, `gkMPI_Barrier`, `gkMPI_Allreduce`).
*   **Other Interactions**:
    *   The output dual graph (`*r_xadj`, `*r_adjncy`) uses `elmdist` as its vertex distribution. Each element of the original mesh becomes a vertex in the dual graph.
    *   The user is responsible for freeing the allocated `*r_xadj` and `*r_adjncy` using `METIS_Free` (as ParMETIS uses `malloc` for these outputs, aligning with METIS conventions for graph structures returned by API calls).
    *   The efficiency of the hash table (`htable` with fixed size `mask+1`) might be a factor for meshes with very high node degrees in the dual (i.e., elements connected to many other elements via shared nodes, though this is less common for typical mesh duals).

```
