# libparmetis/debug.c

## Overview

This file contains various utility functions used for debugging purposes within the ParMETIS library. These functions facilitate the printing of distributed data structures like vectors and graphs, communication setup information, and other internal states in a synchronized manner across multiple MPI processes. They are primarily used during development and testing to verify the correctness of algorithms and data.

## Key Components

### Functions/Classes/Modules

*   **`PrintVector(ctrl_t *ctrl, idx_t n, idx_t first, idx_t *vec, char *title)`**:
    *   Prints an array (`vec`) of `n` elements from each processor. `first` is an offset for global indexing in the printout. Output is synchronized by PE.
*   **`PrintVector2(ctrl_t *ctrl, idx_t n, idx_t first, idx_t *vec, char *title)`**:
    *   Similar to `PrintVector`, but seems to interpret `vec` elements with a `KEEP_BIT`, printing a primary value and a secondary value derived from the bit.
*   **`PrintPairs(ctrl_t *ctrl, idx_t n, ikv_t *pairs, char *title)`**:
    *   Prints an array of `n` `ikv_t` (key-value) pairs from each processor. Output is synchronized.
*   **`PrintGraph(ctrl_t *ctrl, graph_t *graph)`**:
    *   Prints the local portion of a distributed graph (`graph`) from each processor, including vertex weights, adjacency lists, and edge weights. Output is synchronized.
*   **`PrintGraph2(ctrl_t *ctrl, graph_t *graph)`**:
    *   Similar to `PrintGraph`, but also includes partitioning information (`where[i]`) and refinement data (`ckrinfo[i].id`, `ckrinfo[i].ed`) for each vertex.
*   **`PrintSetUpInfo(ctrl_t *ctrl, graph_t *graph)`**:
    *   Prints detailed information about the communication setup for a distributed graph, including send/receive counts and indices for each neighboring processor. Output is synchronized.
*   **`PrintTransferedGraphs(ctrl_t *ctrl, idx_t nnbrs, idx_t *peind, idx_t *slens, idx_t *rlens, idx_t *sgraph, idx_t *rgraph)`**:
    *   Prints information about graphs (or subgraphs) that are being transferred between processors, detailing vertex counts, edge counts, and adjacency information for sent and received graph portions.
*   **`WriteMetisGraph(idx_t nvtxs, idx_t *xadj, idx_t *adjncy, idx_t *vwgt, idx_t *adjwgt)`**:
    *   Writes a graph (assumed to be local, from a single processor's perspective) to a file named "test.graph" in the standard METIS graph file format. This is useful for extracting a local graph component for analysis with serial METIS tools.

## Important Variables/Constants

*   **`ctrl->mype`**: MPI rank of the current processor, used for coordinating print order.
*   **`ctrl->npes`**: Total number of MPI processors.
*   **`ctrl->comm`**: MPI communicator used for `gkMPI_Barrier` to synchronize output.
*   **`graph->vtxdist`**: Vertex distribution array, used to get `firstvtx` for global numbering in printouts.
*   **`KEEP_BIT`**: A bitmask (likely `(1<<30)` or similar, from context of its usage with `idx_t`) used in `PrintVector2` to encode two pieces of information in a single `idx_t`.

## Usage Examples

```c
// Example of how PrintVector might be used internally:
// if (ctrl->dbglvl & DBG_SOME_FLAG) {
//   PrintVector(ctrl, graph->nvtxs, graph->vtxdist[ctrl->mype], graph->where, "Partition Vector");
// }

// To write the local graph of PE 0 to "test.graph":
// if (ctrl->mype == 0) {
//   WriteMetisGraph(graph->nvtxs, graph->xadj, graph->adjncy, graph->vwgt, graph->adjwgt);
// }
```
These functions are typically called within `IFSET(ctrl->dbglvl, DBG_XYZ, ...)` blocks, meaning they are active only when specific debug flags are enabled in the `ctrl->dbglvl` parameter.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides `ctrl_t`, `graph_t`, `ikv_t` definitions, `idx_t` type, `KEEP_BIT` constant (likely via included `defs.h` or `struct.h`), and MPI wrappers (`gkMPI_Barrier`).
    *   These functions are not typically depended upon by core algorithmic modules but are rather called from them for debugging.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: `gkMPI_Barrier` is used extensively to ensure that output from different processors is printed in an orderly fashion rather than interleaved.
    *   Standard C I/O: `fprintf`, `printf`, `fflush`, `fopen`, `fclose`.
*   **Other Interactions**:
    *   The output of these functions is directed to `stdout`.
    *   `WriteMetisGraph` creates a file named "test.graph" in the current working directory.

```
