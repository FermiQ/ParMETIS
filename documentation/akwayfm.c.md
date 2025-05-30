# libparmetis/akwayfm.c

## Overview

This file implements the adaptive k-way refinement algorithm for parallel graph partitioning. Its primary role within the ParMETIS library is to improve the quality of an existing k-way partition by iteratively moving vertices between partitions to reduce edge cut while maintaining load balance. The "adaptive" nature likely refers to how it adjusts its strategy based on the current state of the partition, possibly by focusing on specific types of vertices or regions.

## Key Components

### Functions/Classes/Modules

*   **`KWayAdaptiveRefine(ctrl_t *ctrl, graph_t *graph, idx_t npasses)`**:
    *   This is the main function in this file. It performs k-way partition refinement.
    *   `ctrl`: A control structure containing global parameters like the number of processing elements (PEs), current PE ID, number of partitions, MPI communicator, and various tuning parameters (e.g., `ipc_factor`, `redist_factor`, debug levels).
    *   `graph`: A graph structure containing local graph information (number of vertices/edges, adjacency lists, vertex weights, current partition assignments (`where` array), partition weights, etc.) and information about its distribution across PEs.
    *   `npasses`: The maximum number of refinement passes to perform.
    *   The function iterates multiple passes, and in each pass:
        1.  It potentially permutes the order of processing partitions or vertices.
        2.  It checks for load imbalance.
        3.  It iterates through local vertices (in two sub-passes, potentially prioritizing "dirty" vertices or based on a `ProperSide` macro).
        4.  For each border vertex, it evaluates potential moves to neighboring partitions based on a gain function (considering inter-processor communication (IPC) gain and redistribution gain) and load balance constraints (`badmaxpwgt`).
        5.  If a beneficial move is found, vertex partition assignment (`tmp_where`) and local/global partition weights (`lnpwgts`, `gnpwgts`) are provisionally updated. Vertex degree information (`ckrinfo`) is also updated.
        6.  After an initial pass of proposing moves, it performs a global reduction (`gkMPI_Allreduce`) to get proposed global partition weights (`pgnpwgts`).
        7.  It calculates an `overfill` factor to determine if any proposed moves violate balance constraints excessively.
        8.  If overweight, it iterates through the moved vertices again, potentially reverting some moves that contribute most to imbalance or have low gain, updating weights and degree information accordingly.
        9.  Commits the final set of moves for the current sub-pass by updating the primary `where` array.
        10. Identifies vertices whose external degree information needs to be updated locally (`update` array) or on other PEs (`supdate` array).
        11. Communicates changed interface data (`CommChangedInterfaceData`) and lists of vertices needing updates to neighboring PEs using MPI (`gkMPI_Irecv`, `gkMPI_Isend`, `gkMPI_Waitall`).
        12. Updates the external degree information (`ckrinfo`, `oldEDs`) for local and received remote vertices.
        13. Updates the global partition weights (`gnpwgts`) and the total cut (`graph->mincut`).
        14. The process stops if the cut does not improve or `npasses` are completed.

### Macros

*   **`ProperSide(c, from, other)`**:
    *   A macro likely used to determine if a move from partition `from` to partition `other` is favorable in the current sub-pass `c`. It seems to divide partitions into two conceptual sides based on a permutation `pperm` and `c` (0 or 1) determines which direction of moves are currently being considered.

## Important Variables/Constants

*   **Graph Structure & Partitioning Data**:
    *   `graph->nvtxs`, `graph->nedges`: Number of local vertices and edges.
    *   `graph->vtxdist`: Distribution of vertices across PEs.
    *   `graph->xadj`, `graph->adjncy`, `graph->adjwgt`: Local graph structure (CSR format).
    *   `graph->where`: Array storing the partition assignment for each local and ghost vertex.
    *   `graph->lnpwgts`, `graph->gnpwgts`: Local and global partition weights for each constraint.
    *   `graph->ncon`: Number of constraints per vertex.
    *   `graph->nvwgt`: Weights of vertices for each constraint.
    *   `graph->vsize`: Size of the vertex (used for redistribution gain).
    *   `graph->ckrinfo`: Array of `ckrinfo_t` structs, storing refinement information for each vertex (internal degree `id`, external degree `ed`, neighbor partition info `inbr`, `nnbrs`).
    *   `graph->mincut`, `graph->lmincut`: Global and local edge cut.
    *   `graph->home`: Array indicating the original PE of a vertex (used in uncoupled refinement).
*   **Control & MPI Data**:
    *   `ctrl->npes`, `ctrl->mype`: Total number of PEs and current PE's ID.
    *   `ctrl->nparts`: Total number of partitions.
    *   `ctrl->comm`: MPI communicator.
    *   `ctrl->ubvec`: Target upper bound for partition weights (balance constraint).
    *   `ctrl->tpwgts`: Target partition weights.
    *   `ctrl->ipc_factor`, `ctrl->redist_factor`: Factors controlling the importance of edge cut reduction vs. load balance/redistribution.
    *   `graph->nnbrs`, `graph->peind`, `graph->recvptr`, `graph->sendptr`: Data for MPI communication setup (neighboring PEs, send/receive buffer pointers).
*   **Temporary & Algorithm State Variables**:
    *   `npasses`: Number of refinement passes.
    *   `tmp_where`: Temporary array for storing proposed partition assignments during a pass.
    *   `moved`: Array to store indices of vertices moved in a pass.
    *   `perm`, `pperm`: Permutation arrays for randomizing vertex and partition processing order.
    *   `update`, `supdate`, `rupdate`: Arrays to manage lists of vertices whose states (e.g., degrees) need updating locally or remotely.
    *   `htable`: Hash table to quickly check if a vertex is already marked for update.
    *   `oldEDs`: Stores external degrees of vertices before an inner refinement iteration to track changes to `lmincut`.
    *   `badmaxpwgt`: Computed maximum allowed weight for each partition based on `ubvec` and `tpwgts`.
    *   `movewgts`: Accumulates the sum of weights of vertices moved to/from partitions in a pass.
    *   `ognpwgts`: Stores global partition weights at the beginning of a sub-pass.
    *   `pgnpwgts`: Stores proposed global partition weights after a round of moves.
    *   `overfill`: Array to quantify how much proposed moves violate balance constraints.
    *   `swchanges`, `rwchanges`: Buffers for sending/receiving information about interface vertex changes.
*   **Constants (Implicit)**:
    *   `SMALLFLOAT`: A small floating-point value used for comparisons, likely defined in `parmetislib.h` or a related header.

## Usage Examples

```
N/A
```
This function is an internal component of the ParMETIS library and is typically called as part of a larger partitioning or repartitioning process (e.g., by functions like `ParMETIS_V3_AdaptiveRepart`). It's not meant to be used directly by end-users in standalone code.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: This header file is directly included. It provides definitions for core data structures (`ctrl_t`, `graph_t`, `ckrinfo_t`, `cnbr_t`, `ikv_t`), type definitions (`idx_t`, `real_t`), macros (e.g., `IFSET`, `DBG_TIME`, `WCOREPUSH`/`POP`, `PASSERT`, `INC_DEC`, `gk_SWAP`, `gk_max`, `rabs`), MPI wrapper function prototypes (`gkMPI_*`), and likely prototypes for various utility and helper functions used within `KWayAdaptiveRefine` such as:
        *   `RandomPermute`, `FastRandomPermute`: For permuting arrays.
        *   `ComputeParallelBalance`: To check load balance.
        *   `ravg`: To compute average of real values.
        *   `IsHBalanceBetterTT`, `IsHBalanceBetterFT`: To compare balance improvements.
        *   `cnbrpoolGetNext`: To get memory from a pool for neighbor info.
        *   `CommChangedInterfaceData`: For communicating interface changes.
        *   `GlobalSESum`: For summing values across PEs.
        *   `isorti`: For sorting integer arrays.
        *   `rprintf`: For parallel printing (rank 0).
        *   Memory management (`rwspacemalloc`, `iwspacemalloc`, `ikvwspacemalloc`, `iset`, `icopy`, `rcopy`, `rset`).
        *   Timers (`starttimer`, `stoptimer`).
    *   Other C files in ParMETIS: The function calls various helper and MPI communication wrapper functions that are implemented in other `.c` files within the `libparmetis` directory (e.g., functions related to communication, graph data structure manipulation, memory management, utility functions).
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Used extensively for parallel communication, as indicated by functions like `gkMPI_Allreduce`, `gkMPI_Bcast`, `gkMPI_Irecv`, `gkMPI_Isend`, `gkMPI_Waitall`, `gkMPI_Get_count`, and the use of `MPI_SUM`.
*   **Other Interactions**:
    *   The function directly modifies the `graph_t` structure passed to it, particularly `graph->where` (partition assignments), `graph->lnpwgts` and `graph->gnpwgts` (partition weights), `graph->ckrinfo` (refinement information), and `graph->mincut` (edge cut).
    *   It reads control parameters and MPI settings from the `ctrl_t` structure.
    *   The algorithm's behavior is influenced by the initial state of the graph partition and the various parameters set in `ctrl_t`.

```
