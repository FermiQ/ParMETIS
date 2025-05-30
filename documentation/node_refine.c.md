# libparmetis/node_refine.c

## Overview

This file contains routines specifically designed for k-way node-based refinement, primarily used in the context of parallel nested dissection ordering (`ometis.c`). Unlike edge-based refinement which focuses on edge cuts, node-based refinement typically aims to minimize the size of the vertex separator. These routines manage data structures for node refinement (`NRInfoType`), compute parameters like external degrees to different partitions, and implement greedy refinement strategies.

## Key Components

### Functions/Classes/Modules

*   **`AllocateNodePartitionParams(ctrl_t *ctrl, graph_t *graph)`**:
    *   Allocates memory for data structures required for node-based refinement.
    *   `graph->nrinfo` (array of `NRInfoType`): Stores refinement information for each node, particularly its external degrees to different sides of a bisection/multisection.
    *   `graph->lpwgts`, `graph->gpwgts`: Local and global partition weights (often, 0 for part A, 1 for part B, and 2+ for separators).
    *   `graph->sepind`: Array to store indices of separator vertices.
    *   Reallocates `graph->vwgt` to include space for ghost vertex weights.
*   **`ComputeNodePartitionParams(ctrl_t *ctrl, graph_t *graph)`**:
    *   Computes initial parameters for node refinement. Assumes `AllocateNodePartitionParams` has been called.
    *   Communicates `where` (partition assignments) and `vwgt` (vertex weights) for interface vertices.
    *   For each local vertex:
        *   Updates `lpwgts` based on its `where` and `vwgt`.
        *   If it's a separator vertex (`where[i] >= nparts`):
            *   Adds it to `sepind`.
            *   Calculates `rinfo[i].edegrees[0]` and `rinfo[i].edegrees[1]`, which are the sum of `vwgt` of its neighbors in partition 0 and partition 1 (or `part_A_of_sep_i` and `part_B_of_sep_i`).
    *   Updates `graph->nsep` (number of local separator vertices).
    *   Computes global partition weights `gpwgts` and `graph->mincut` (total weight of separator vertices).
*   **`UpdateNodePartitionParams(ctrl_t *ctrl, graph_t *graph)`**:
    *   Similar to `ComputeNodePartitionParams`, but called after some vertices might have changed their `where` status (e.g., moved into or out of a separator). Recomputes `lpwgts`, `gpwgts`, `sepind`, `nsep`, and `edegrees` for separator nodes.
*   **`KWayNodeRefine_Greedy(ctrl_t *ctrl, graph_t *graph, idx_t npasses, real_t ubfrac)`**:
    *   Performs k-way node-based greedy refinement.
    *   Aims to move separator nodes into one of the partitions if doing so is beneficial (reduces overall "cost" or improves balance) and maintains balance constraints (`badmaxpwgt`).
    *   Iterates for `npasses` or until no improvement. Each pass has two `side` iterations (0 and 1).
    *   Uses a priority queue (`rpq_t`) to select separator nodes to move. The key for a node `i` in the queue is `vwgt[i] - rinfo[i].edegrees[cc]`, where `cc` is the side opposite to the target move side `c`. This prioritizes nodes with small external degree to the non-target side.
    *   If a move is made:
        *   Updates `where`, `lpwgts`, `gpwgts`.
        *   Identifies neighboring vertices whose `edegrees` might change and updates them in the priority queue or marks them for broader update.
        *   Communicates changes to `where` for interface vertices (`CommChangedInterfaceData`) and lists of vertices needing `edegree` updates.
    *   The refinement alternates between trying to move nodes to partition `c` and then to partition `cc`.
*   **`KWayNodeRefine2Phase(ctrl_t *ctrl, graph_t *graph, idx_t npasses, real_t ubfrac)`**:
    *   A two-phase refinement strategy.
    *   Repeatedly calls `KWayNodeRefine_Greedy` and then `KWayNodeRefineInterior`.
    *   After each call, it updates partition parameters (`UpdateNodePartitionParams`).
    *   Stops if no improvement in `graph->mincut` (separator size).
*   **`KWayNodeRefineInterior(ctrl_t *ctrl, graph_t *graph, idx_t npasses, real_t ubfrac)`**:
    *   Refines interior nodes (non-interface nodes) of each local "subgraph" (formed by a pair of partitions and their separator, e.g., `gid`, `gid+1`, `nparts+gid`).
    *   It extracts these local subgraphs, converts them to a format suitable for serial METIS node refinement (`METIS_NodeRefine`), calls it, and then maps the results back to the global `graph->where` array.
*   **`PrintNodeBalanceInfo(ctrl_t *ctrl, idx_t nparts, idx_t *gpwgts, idx_t *badmaxpwgt, char *title)`**:
    *   A utility function to print balance information related to node-based partitioning (weights of parts 0, 1, and their separator, against `badmaxpwgt`).

## Important Variables/Constants

*   **`graph->nrinfo` (array of `NRInfoType`)**: Stores `edegrees[2]` for separator nodes, representing sum of neighbor weights in partition 0 and 1 relative to the separator.
*   **`graph->lpwgts`, `graph->gpwgts`**: Partition weights. For node refinement, indices `0` to `nparts-1` usually refer to the actual partitions, and `nparts` to `2*nparts-1` might refer to separators, with `2*nparts-1` often being a total separator weight.
*   **`graph->sepind`**: Array storing indices of local separator vertices.
*   **`graph->nsep`**: Number of local separator vertices.
*   **`badmaxpwgt`**: Maximum allowed weight for partitions, derived from `ubfrac` and current partition weights.
*   `queue (rpq_t *)`: Priority queue used in `KWayNodeRefine_Greedy`.

## Usage Examples

```
N/A
```
These are internal ParMETIS functions, primarily called by the parallel nested dissection ordering algorithm (`ometis.c`).
*   `AllocateNodePartitionParams` and `ComputeNodePartitionParams` (or `UpdateNodePartitionParams`) are called to initialize or refresh data needed for refinement.
*   `KWayNodeRefine_Greedy` or `KWayNodeRefine2Phase` are then called to perform the actual refinement of the vertex separators.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Core definitions (`ctrl_t`, `graph_t`, `NRInfoType`, `idx_t`, `rpq_t`), MPI wrappers, memory utilities, GKlib priority queue functions (`rpqCreate`, `rpqInsert`, etc.).
    *   `comm.c`: `CommInterfaceData`, `CommChangedInterfaceData`, `GlobalSESum`.
    *   `ometis.c`: These functions are central to the refinement steps within the multilevel ordering process.
    *   `metis.h` (external METIS library): `METIS_NodeRefine` is called by `KWayNodeRefineInterior`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: For communication.
    *   `METIS`: Serial METIS library for interior node refinement.
*   **Other Interactions**:
    *   These routines modify `graph->where` by moving nodes into or out of separators, and update `graph->mincut` (total separator size).
    *   The refinement process is sensitive to the `ubfrac` (imbalance tolerance) parameter.

```
