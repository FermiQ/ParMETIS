# libparmetis/struct.h

## Overview

This header file defines the core internal data structures used throughout the ParMETIS library. These structures are essential for representing graphs, meshes, control parameters, refinement information, and other auxiliary data needed by the various parallel graph partitioning and ordering algorithms.

## Key Components

### Functions/Classes/Modules/Macros/Structs

This file primarily consists of `typedef struct` definitions.

*   **`cnbr_t`**:
    *   Stores information about a neighboring partition for a given vertex during cut-based k-way refinement.
    *   `pid`: The partition ID of the neighbor.
    *   `ed`: The sum of weights of edges connecting the vertex to this neighboring partition.
*   **`i2kv_t`**:
    *   A simple structure to store a triplet of `idx_t` values: `key1`, `key2`, and `val`. Used for sorting or storing multi-keyed data.
*   **`ckrinfo_t`**:
    *   Stores cut-based k-way refinement information for a vertex.
    *   `id`: Internal degree (sum of weights of edges to vertices in the same partition).
    *   `ed`: External degree (sum of weights of edges to vertices in different partitions).
    *   `nnbrs`: Number of distinct neighboring partitions.
    *   `inbr`: Index into a global pool (`ctrl->cnbrpool`) where the list of `cnbr_t` for this vertex is stored.
*   **`NRInfoType` (and `nrinfodef`)**:
    *   Stores node-refinement information, specifically for 2-way (bisection/multisection) scenarios.
    *   `edegrees[2]`: Stores the sum of vertex weights of neighbors in partition 0 and partition 1, respectively, for a separator node.
*   **`matrix_t`**:
    *   Represents a sparse matrix, often used for connectivity graphs between partitions in diffusion algorithms.
    *   `nrows`, `nnzs`: Number of rows and non-zeros.
    *   `rowptr`, `colind`: CSR row pointers and column indices.
    *   `values`: Matrix values (often diagonal elements store degree/self-loop, off-diagonals are negative).
    *   `transfer`: Array to store flow/transfer values associated with matrix edges (non-zeros).
*   **`graph_t`**:
    *   The main structure for representing a distributed graph. It's a complex structure containing:
        *   Basic graph info: `gnvtxs` (global vertices), `nvtxs` (local vertices), `nedges`, `ncon` (number of constraints).
        *   CSR data: `xadj`, `adjncy`, `adjwgt`.
        *   Vertex data: `vwgt` (weights), `nvwgt` (normalized weights), `vsize` (sizes for redistribution).
        *   Distribution: `vtxdist`.
        *   Partitioning info: `home` (initial partition), `where` (current partition).
        *   Memory ownership flags: `free_xadj`, `free_adjncy`, etc.
        *   Coarsening info: `match`, `cmap`.
        *   Communication setup: `nnbrs`, `nrecv`, `nsend`, `peind`, `sendptr`/`ind`, `recvptr`/`ind`, `imap`, `pexadj`/`peadjncy`/`peadjloc`, `lperm`.
        *   Projection info: `rlens`, `slens`, `rcand`.
        *   Refinement info: `lpwgts`/`gpwgts` (partition weights, older style), `lnpwgts`/`gnpwgts` (normalized partition weights), `ckrinfo` (for k-way edge refinement), `nrinfo` (for node refinement), `sepind`.
        *   Cut info: `lmincut`, `mincut`.
        *   Multilevel hierarchy: `level`, `coarser`, `finer` pointers.
        *   On-disk processing: `gID`, `ondisk`.
*   **`timer` (typedef double)**: Defines the type used for timing various operations.
*   **`ctrl_t`**:
    *   The main control structure, holding global parameters and state for a ParMETIS operation.
    *   Operation type: `optype`.
    *   MPI info: `mype`, `npes`, `gcomm`, `comm`, request arrays.
    *   Partitioning parameters: `ncon`, `nparts`, `CoarsenTo`, `tpwgts`, `ubvec`, `ubfrac`.
    *   Algorithm choices: `mtype` (matching), `ipart` (initial partitioning), `rtype` (refinement).
    *   Debugging: `dbglvl`, `seed`.
    *   Factors: `redist_factor`, `ipc_factor`.
    *   Workspace: `mcore` (GKlib memory core), `cnbrpool` and related fields for refinement neighbor lists.
    *   On-disk info: `ondisk`, `pid`.
    *   Timers: Numerous `timer` fields for profiling different stages.
*   **`mesh_t`**:
    *   Represents a distributed mesh.
    *   `etype`: Element type.
    *   `gnelms`, `nelms`: Global/local number of elements.
    *   `gnns`: Global number of nodes.
    *   `ncon`: Number of weights per element.
    *   `esize`: Number of nodes per element.
    *   `gminnode`: Global minimum node ID (for normalization).
    *   `elmdist`: Distribution of elements.
    *   `elements`: Element connectivity array.
    *   `elmwgt`: Element weights.

## Important Variables/Constants

This file defines the *structure* of these variables. The actual instances and their values are managed by other parts of the library. Key concepts embodied by members:
*   CSR representation for graphs and sparse matrices.
*   Distinction between local and global vertex/element counts and indices.
*   Storage for various weights and costs (vertex weights, edge weights, vertex sizes, partition weights).
*   Pointers for building multilevel hierarchies (`coarser`, `finer`).
*   Flags for memory management (`free_*`).
*   Comprehensive communication buffers and metadata within `graph_t`.
*   Detailed control parameters within `ctrl_t`.

## Usage Examples

```c
// This is a header file. It's included by parmetislib.h, which is then
// included by most .c files in the ParMETIS library.

// Example: In graph.c
// #include <parmetislib.h> // This will include struct.h
//
// graph_t *mygraph = CreateGraph(); // CreateGraph uses sizeof(graph_t)
// mygraph->nvtxs = 100;
// mygraph->xadj = imalloc(101, "xadj_for_mygraph");
// // etc.

// In kmetis.c
// #include <parmetislib.h>
//
// void SomeFunction(ctrl_t *ctrl, graph_t *graph) {
//   if (ctrl->dbglvl & DBG_INFO) { ... }
//   idx_t nnbrs = graph->nnbrs;
//   // ...
// }
```

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Includes this file.
    *   `parmetis.h` (via `parmetislib.h`): Provides `idx_t`, `real_t`, `pmoptype_et`, `MPI_Comm`, `MPI_Request`, `MPI_Status`.
    *   `GKlib.h` (via `parmetislib.h`): Provides `ikv_t` (though ParMETIS also defines its own `ikv_t` via `gklib_defs.h` which might be slightly different or just namespaced), `gk_mcore_t`.
    *   Nearly all `.c` files in `libparmetis` depend on these structure definitions to operate on graph data and control parameters.
*   **External Libraries**:
    *   `MPI`: MPI types are used in `ctrl_t`.
*   **Other Interactions**:
    *   These data structures are fundamental to the entire ParMETIS library. Changes to them would have widespread impact.
    *   The design of `graph_t` and `ctrl_t` reflects the complexity of parallel multilevel graph algorithms, encompassing data for the graph itself, its distribution, partitioning state, refinement aids, communication buffers, and control knobs for algorithm behavior.

```
