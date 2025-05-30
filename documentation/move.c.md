# libparmetis/move.c

## Overview

This file contains functions related to redistributing a ParMETIS graph (`graph_t`) according to a new partition, and projecting information (like partition assignments) back from a redistributed (moved) graph to the original graph's distribution. These are key operations in parallel graph partitioning, especially when an algorithm computes a new partition on a graph that might have a different distribution than the original input, or when the graph data itself needs to be moved to match a new partition.

## Key Components

### Functions/Classes/Modules

*   **`MoveGraph(ctrl_t *ctrl, graph_t *graph)`**:
    *   Redistributes the input `graph` data across processors according to the current partition assignments stored in `graph->where`.
    *   It assumes `ctrl->nparts` (number of target partitions in `graph->where`) is less than or equal to `ctrl->npes` (number of processors), implying each target partition will reside on at most one processor.
    *   **Steps**:
        1.  Calculates `mvtxdist`: the vertex distribution of the new "moved" graph (`mgraph`). Each PE `p` will own vertices assigned to partition `p`.
        2.  Determines `newlabel` for each local vertex: its new global index in `mgraph`. This involves a parallel prefix scan (`gkMPI_Scan`) on the counts of vertices going to each PE.
        3.  Communicates `newlabel` for interface vertices (`CommInterfaceData`).
        4.  Each PE determines how many vertices and edges it's sending to every other PE (`sinfo`) and how many it's receiving (`rinfo`) using `gkMPI_Alltoall`.
        5.  Graph data (CSR structure, vertex weights) is packed into `sgraph` and sent to target PEs using non-blocking sends/receives (`gkMPI_Isend`, `gkMPI_Irecv`, `gkMPI_Waitall`). Adjacency lists in `sgraph` use the `newlabel` for adjacencies.
        6.  A new graph `mgraph` is created on each PE from the received data (`rgraph`). `mgraph->vtxdist` is set to `mvtxdist`.
    *   Returns the new `mgraph`.
*   **`ProjectInfoBack(ctrl_t *ctrl, graph_t *graph, idx_t *info, idx_t *minfo)`**:
    *   Transfers information from an array `minfo` (which is distributed according to a "moved" graph, likely the result of `MoveGraph` or a similar operation) back to an array `info` that corresponds to the original distribution of `graph`.
    *   `graph->where` is assumed to hold the `npes`-way partition that describes how `graph` was moved to create the graph associated with `minfo`.
    *   **Steps**:
        1.  Each PE determines how many elements of `minfo` it needs to send to other PEs based on `graph->where` (which indicates the original PE of the vertices that were moved to the current PE).
        2.  Uses `gkMPI_Alltoall` to communicate these counts.
        3.  Uses non-blocking sends/receives to transfer relevant portions of `minfo` to the PEs that need them (those PEs are the original owners of the vertices).
        4.  Received data is scattered into the `info` array according to `graph->where`.
*   **`FindVtxPerm(ctrl_t *ctrl, graph_t *graph, idx_t *perm)`**:
    *   Computes a permutation array `perm` based on the current partition assignments in `graph->where`.
    *   The permutation orders vertices by their partition ID, and within each partition, by their original relative order on their source PE.
    *   This is useful for reordering vertex data to be contiguous by partition.
    *   Uses `gkMPI_Scan` and `gkMPI_Allreduce` to determine global offsets for each partition.
*   **`CheckMGraph(ctrl_t *ctrl, graph_t *graph)`**:
    *   A debugging function to check the consistency of a moved graph.
    *   It verifies that if vertex `u` (local to current PE) has an edge to local vertex `v`, then `v` also has an edge back to `u`.
    *   Prints error messages if inconsistencies are found.

## Important Variables/Constants

*   **`graph->where`**: Array storing the partition assignment for each vertex. Crucial for both `MoveGraph` (determines destination PE) and `ProjectInfoBack` (determines source PE of data).
*   **`newlabel` (in `MoveGraph`)**: Temporary array storing the new global indices of vertices after re-distribution.
*   **`mvtxdist` (in `MoveGraph`)**: The `vtxdist` for the newly created `mgraph`.
*   **`sinfo`, `rinfo` (in `MoveGraph`, `ProjectInfoBack`)**: Arrays of `ikv_t` (or similar) used to store counts of vertices/edges/data being sent/received between PEs.
*   **`sgraph`, `rgraph` (in `MoveGraph`)**: Buffers for sending and receiving graph data during redistribution.

## Usage Examples

```
N/A
```
These are internal ParMETIS functions.
*   `MoveGraph` is used when a partitioning algorithm has determined a new distribution of the graph across processors (e.g., after an initial geometric partition in `gkmetis.c`). The graph data is then physically moved to align with this new distribution for subsequent processing.
*   `ProjectInfoBack` is used after computations (like partitioning) are done on a moved/redistributed graph, to transfer results (e.g., final partition vector) back to the original data distribution.
*   `FindVtxPerm` might be used to prepare data for remapping or if a contiguous ordering by partition is needed.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Core definitions (`ctrl_t`, `graph_t`, `idx_t`, `ikv_t`), MPI wrappers, memory utilities.
    *   `graph.c`: For `CreateGraph`.
    *   `comm.c`: For `CommInterfaceData`, `CommUpdateNnbrs`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Extensively used for point-to-point and collective communication (`gkMPI_Scan`, `gkMPI_Allreduce`, `gkMPI_Alltoall`, `gkMPI_Isend`, `gkMPI_Irecv`, `gkMPI_Waitall`).
*   **Other Interactions**:
    *   `MoveGraph` results in a new `graph_t` structure (`mgraph`) with potentially different local `nvtxs` on each PE compared to the input `graph`.
    *   The correctness of `ProjectInfoBack` relies on `graph->where` accurately reflecting how the data in `minfo` was distributed from the original `graph`.

```
