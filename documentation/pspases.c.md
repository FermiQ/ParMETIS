# libparmetis/pspases.c

## Overview

This file contains routines specifically designed for interfacing ParMETIS with PSPASES, a parallel sparse direct solver library. The primary function `ParMETIS_SerialNodeND` provides a nested dissection ordering by first assembling the entire distributed graph onto a single processor (PE 0) and then calling the serial METIS nested dissection routine (`METIS_NodeNDP`) on it. The resulting ordering and separator sizes are then scattered back to all processors.

## Key Components

### Functions/Classes/Modules

*   **`ParMETIS_SerialNodeND(idx_t *vtxdist, idx_t *xadj, idx_t *adjncy, idx_t *numflag, idx_t *options, idx_t *order, idx_t *sizes, MPI_Comm *comm)`**:
    *   This is an API-level function, likely intended for use when a serial nested dissection ordering of the entire graph is desired, possibly for comparison or when PSPASES is used in a mode that benefits from such an ordering.
    *   **Functionality**:
        1.  Sets up a ParMETIS control structure (`ctrl_t`).
        2.  Checks if the number of processors (`npes`) is a power of 2, which is a requirement for `METIS_NodeNDP`.
        3.  If `numflag > 0` (1-based input), converts graph numbering to 0-based (`ChangeNumbering`).
        4.  Calls `AssembleEntireGraph` to gather the full distributed graph onto PE 0. The assembled graph is `agraph`.
        5.  **On PE 0 only**:
            *   Allocates `perm` and `iperm` arrays for the ordering.
            *   Calls `METIS_NodeNDP` (a METIS function, possibly specialized for creating orderings suitable for PSPASES or that outputs `npes` separator sizes) on `agraph` to compute the nested dissection ordering (`iperm`) and separator tree structure (`sizes`).
        6.  Broadcasts the `sizes` array from PE 0 to all other processors.
        7.  Scatters the computed ordering (`iperm` from PE 0) to the `order` array on each processor, distributing the relevant portions based on `vtxdist`.
        8.  Frees allocated memory.
        9.  If `numflag > 0`, converts the `order` array back to 1-based numbering.
*   **`AssembleEntireGraph(ctrl_t *ctrl, idx_t *vtxdist, idx_t *xadj, idx_t *adjncy)`**:
    *   Gathers the entire distributed graph (CSR structure: `xadj` and `adjncy`) onto processor 0.
    *   **Steps**:
        1.  Each processor first sends its local `xadj` degree counts to PE 0 using `gkMPI_Gatherv`. PE 0 reconstructs the global `axadj` (CSR row pointers for the assembled graph).
        2.  Each processor sends its local `adjncy` (adjacency lists) to PE 0 using `gkMPI_Gatherv`. PE 0 reconstructs the global `aadjncy`.
    *   Returns a `graph_t` structure (`agraph`) on PE 0 containing the assembled graph. On other PEs, `agraph` might be created but would not contain the full graph data. (Note: The implementation seems to allocate `axadj` and `aadjncy` on all PEs but only PE0 fills them with the full graph).

## Important Variables/Constants

*   **`agraph` (graph_t)**: Graph structure holding the entire assembled graph, primarily on PE 0.
*   **`perm`, `iperm` (in `ParMETIS_SerialNodeND` on PE 0)**: Permutation and inverse permutation arrays for the nested dissection ordering. `iperm` stores the actual ordering.
*   **`sizes` (output of `ParMETIS_SerialNodeND`)**: Array storing the sizes of the separators in the nested dissection tree, as defined by `METIS_NodeNDP`.
*   **`order` (output of `ParMETIS_SerialNodeND`)**: Array storing the segment of the global ordering relevant to each PE.

## Usage Examples

```c
// Conceptual call to ParMETIS_SerialNodeND
// (assuming vtxdist, xadj, adjncy, order, sizes, comm are set up)
// idx_t numflag = 0;
// idx_t options[METIS_NOPTIONS]; options[0] = 0; // Default options
//
// ParMETIS_SerialNodeND(vtxdist, xadj, adjncy, &numflag, options,
//                       order, sizes, &comm);
//
// // 'order' now contains the fill-reducing permutation computed serially on PE 0
// // and distributed.
// // 'sizes' contains the separator tree sizes.
```
This is an API function, potentially used in conjunction with the PSPASES solver.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Core definitions.
    *   `ctrl.c`: For `SetupCtrl`, `FreeCtrl`.
    *   `graph.c`: For `CreateGraph`.
    *   `util.c`: For `ChangeNumbering`.
    *   `gkmpi.c`: For MPI wrappers like `gkMPI_Gatherv`, `gkMPI_Bcast`, `gkMPI_Scatterv`.
    *   `metis.h` (external METIS library): `METIS_NodeNDP` is the core serial ordering routine used on PE 0.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Used for gathering the graph and distributing the results.
    *   `METIS`: The serial METIS library is essential for the actual nested dissection ordering.
*   **Other Interactions**:
    *   This function provides a way to obtain a serial nested dissection ordering for a distributed graph, which might be required by certain modes of operation of parallel direct solvers like PSPASES or for comparison purposes.
    *   The requirement that `npes` must be a power of 2 is a specific constraint imposed by `METIS_NodeNDP`.
    *   The majority of the computational work (graph assembly and ordering) happens on PE 0, making it a bottleneck for very large graphs.

```
