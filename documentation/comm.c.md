# libparmetis/comm.c

## Overview

This file provides high-level communication functions essential for parallel graph operations within ParMETIS. It includes routines to set up communication structures for distributed graphs, exchange data for interface (ghost) vertices, and perform global reductions (like sum, min, max) across MPI processes. These functions abstract the underlying MPI calls and manage the necessary data structures for efficient inter-process communication.

## Key Components

### Functions/Classes/Modules

*   **`CommSetup(ctrl_t *ctrl, graph_t *graph)`**:
    *   Sets up the communication infrastructure for a distributed graph.
    *   Determines which processors own vertices adjacent to local vertices (i.e., identifies interface/ghost vertices).
    *   Localizes the numbering of adjacency lists: local vertices are numbered `0` to `nvtxs-1`, and remote (ghost) vertices are numbered `nvtxs` to `nvtxs+nrecv-1`.
    *   Populates graph fields related to communication: `graph->nrecv` (number of ghost vertices), `graph->recvind` (global indices of ghost vertices), `graph->nnbrs` (number of neighbor PEs), `graph->peind` (IDs of neighbor PEs), `graph->recvptr` (CSR-like pointer for `recvind` per PE).
    *   Determines send information: `graph->nsend` (number of local vertices to be sent), `graph->sendind` (local indices of vertices to be sent), `graph->sendptr` (CSR-like pointer for `sendind` per PE).
    *   Creates `graph->pexadj`, `graph->peadjncy`, `graph->peadjloc` for sparse boundary exchanges.
    *   Creates `graph->imap` to map local/ghost indices back to global vertex indices.
*   **`CommUpdateNnbrs(ctrl_t *ctrl, idx_t nnbrs)`**:
    *   Updates (reallocates if necessary) MPI request and status arrays within the `ctrl` structure if the number of communicating neighbors (`nnbrs`) increases beyond the currently allocated capacity (`ctrl->ncommpes`).
*   **`CommInterfaceData(ctrl_t *ctrl, graph_t *graph, idx_t *data, idx_t *recvvector)`**:
    *   Exchanges data for interface/ghost vertices. Each PE sends its local `data` values for vertices listed in `graph->sendind` to appropriate neighbors.
    *   Received data is stored in `recvvector`, corresponding to the order in `graph->recvind`.
    *   Uses non-blocking MPI sends (`gkMPI_Isend`) and receives (`gkMPI_Irecv`) followed by `gkMPI_Waitall`.
*   **`CommChangedInterfaceData(ctrl_t *ctrl, graph_t *graph, idx_t nchanged, idx_t *changed, idx_t *data, ikv_t *sendpairs, ikv_t *recvpairs)`**:
    *   A more optimized version of `CommInterfaceData` for exchanging data of only a subset of interface vertices that have changed.
    *   `nchanged` and `changed` specify the local vertices whose `data` has changed.
    *   Uses `ikv_t` (key-value pairs) to send data, where key is the remote index (offset) and value is the data.
    *   `peadjloc` (from `CommSetup`) is used to map local vertex changes to the correct indices in neighbor PEs' receive buffers.
*   **`GlobalSEMax(ctrl_t *ctrl, idx_t value)`**: Computes the global maximum of an `idx_t` value across all PEs in `ctrl->comm`.
*   **`GlobalSEMaxComm(MPI_Comm comm, idx_t value)`**: Computes global maximum on a specific communicator.
*   **`GlobalSEMin(ctrl_t *ctrl, idx_t value)`**: Computes the global minimum of an `idx_t` value.
*   **`GlobalSEMinComm(MPI_Comm comm, idx_t value)`**: Computes global minimum on a specific communicator.
*   **`GlobalSESum(ctrl_t *ctrl, idx_t value)`**: Computes the global sum of an `idx_t` value.
*   **`GlobalSESumComm(MPI_Comm comm, idx_t value)`**: Computes global sum on a specific communicator.
*   **`GlobalSEMaxFloat(ctrl_t *ctrl, real_t value)`**: Computes the global maximum of a `real_t` value.
*   **`GlobalSEMinFloat(ctrl_t *ctrl, real_t value)`**: Computes the global minimum of a `real_t` value.
*   **`GlobalSESumFloat(ctrl_t *ctrl, real_t value)`**: Computes the global sum of a `real_t` value.

## Important Variables/Constants

*   **`graph->vtxdist`**: Array defining the distribution of global vertices to PEs.
*   **`graph->adjncy`**: Adjacency list; modified by `CommSetup` to use local/ghost numbering.
*   **`graph->lperm`**: Permutation array used internally by `CommSetup` to distinguish local-only vertices.
*   **`graph->nlocal`**: Number of vertices with only local adjacencies.
*   **`graph->imap`**: Maps local vertex indices (0 to nvtxs-1) and ghost vertex indices (nvtxs to nvtxs+nrecv-1) to their global vertex IDs.
*   **`ctrl->sreq`, `ctrl->rreq`, `ctrl->statuses`**: Arrays to manage MPI request handles and statuses for non-blocking communication.

## Usage Examples

```
N/A
```
These are internal ParMETIS communication routines, typically called by other higher-level parallel algorithms (e.g., partitioning, refinement, matrix-vector multiplication). `CommSetup` is fundamental for initializing how a distributed graph communicates. `CommInterfaceData` and `CommChangedInterfaceData` are used to synchronize data associated with ghost vertices. Global reduction functions are used for collective decision-making or data aggregation.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides definitions for `ctrl_t`, `graph_t`, `ikv_t`, type `idx_t`, `real_t`, MPI wrappers (`gkMPI_*`), timer macros (`STARTTIMER`, `STOPTIMER`), memory allocation (`imalloc`, `ikvwspacemalloc`, `iwspacemalloc`, `ismalloc`, `icopy`, `iincset`), CSR utilities (`MAKECSR`, `SHIFTCSR`), sorting (`ikvsorti`), and assertion macros (`PASSERT`, `PASSERTP`).
    *   Other ParMETIS modules: This file is foundational for almost all parallel operations in ParMETIS. Any module that operates on a distributed graph and needs to exchange boundary information or perform global reductions will rely on these communication primitives.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Used for all inter-processor communication (e.g., `MPI_Alltoall`, `MPI_Irecv`, `MPI_Isend`, `MPI_Waitall`, `MPI_Allreduce`). The `gkMPI_` functions are wrappers around standard MPI calls.
*   **Other Interactions**:
    *   `CommSetup` significantly modifies the `graph_t` structure by populating its communication-related fields and renumbering parts of `graph->adjncy`.
    *   The global reduction functions depend on the `ctrl->comm` (or a user-provided `MPI_Comm`) for collective operations.

```
