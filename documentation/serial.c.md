# libparmetis/serial.c

## Overview

This file contains serial (non-parallel) graph partitioning and refinement routines that are used as components within the larger parallel ParMETIS algorithms. These functions typically operate on graph data that has been assembled onto a single processor or is being processed in a serial fashion as part of a more complex parallel scheme (e.g., refining a local subgraph or a graph representing connectivity between partitions). The file includes utilities for computing partition parameters, performing 2-way and k-way refinement, and balancing for serial graphs.

## Key Components

### Functions/Classes/Modules

*   **`Mc_ComputeSerialPartitionParams(ctrl_t *ctrl, graph_t *graph, idx_t nparts)`**:
    *   Computes partition parameters (internal/external degrees, partition weights) for a serial graph `graph` that is partitioned into `nparts`.
    *   Similar to `ComputePartitionParams` but for a non-distributed graph context.
*   **`Mc_SerialKWayAdaptRefine(ctrl_t *ctrl, graph_t *graph, idx_t nparts, idx_t *home, real_t *orgubvec, idx_t npasses)`**:
    *   Performs k-way adaptive refinement on a serial graph.
    *   Uses a greedy approach, sorting vertices by gain (`ed-id`).
    *   Iteratively moves vertices to other partitions if the move improves cut or balance, respecting weight constraints (`maxwgt`, `minwgt` derived from `orgubvec`).
    *   Updates degrees and partition weights after each move.
*   **`AreAllHVwgtsBelow(idx_t ncon, real_t alpha, real_t *vwgt1, real_t beta, real_t *vwgt2, real_t *limit)`**:
    *   Utility to check if a linear combination of vertex weights (`alpha*vwgt1 + beta*vwgt2`) is below a `limit` for all constraints. Used for balance checks.
*   **`ComputeHKWayLoadImbalance(idx_t ncon, idx_t nparts, real_t *npwgts, real_t *lbvec)`**:
    *   Computes the load imbalance for a k-way partition. `lbvec[i]` stores `max_weight_for_constraint_i * nparts`.
*   **`SerialRemap(ctrl_t *ctrl, graph_t *graph, idx_t nparts, idx_t *base, idx_t *scratch, idx_t *remap, real_t *tpwgts)`**:
    *   Remaps a serial k-way partition to minimize redistribution cost, similar to `ParallelTotalVReMap` but for a serial context.
    *   `base` is the initial partition, `scratch` is the target partition (e.g., from a different strategy), `remap` is the output mapping.
    *   Uses a heuristic based on matching heaviest flows between `base` and `scratch` partitions.
*   **`SSMIncKeyCmp(const void *fptr, const void *sptr)`**:
    *   Comparison function for `qsort`, used by `SerialRemap` to sort `i2kv_t` structures (key1 ascending, then key2 descending).
*   **`Mc_Serial_FM_2WayRefine(ctrl_t *ctrl, graph_t *graph, real_t *tpwgts, idx_t npasses)`**:
    *   Performs 2-way FM refinement on a serial graph.
    *   Uses priority queues (`rpq_t`) to select vertices for moves.
    *   Iteratively moves vertices to reduce cut or improve balance (defined by `tpwgts` and `origbal`).
    *   Includes a rollback mechanism to the best state found if moves exceed a `limit` without improvement.
*   **`Serial_SelectQueue(idx_t ncon, real_t *npwgts, real_t *tpwgts, idx_t *from, idx_t *cnum, rpq_t **queues[2])`**:
    *   Selects a queue for 2-way FM based on which partition is most overweight or offers the best gain.
*   **`Serial_BetterBalance(idx_t ncon, real_t *npwgts, real_t *tpwgts, real_t *diff, real_t *tmpdiff)`**:
    *   Checks if the current partition `npwgts` is better balanced towards `tpwgts` than a previous state represented by `diff` (sum of absolute differences).
*   **`Serial_Compute2WayHLoadImbalance(idx_t ncon, real_t *npwgts, real_t *tpwgts)`**:
    *   Computes load imbalance for a 2-way partition (max relative deviation).
*   **`Mc_Serial_Balance2Way(ctrl_t *ctrl, graph_t *graph, real_t *tpwgts, real_t lbfactor)`**:
    *   Balances a 2-way partition to meet `lbfactor` (load balance factor) by moving high-gain vertices.
    *   Uses priority queues and similar logic to FM refinement but prioritizes balance.
*   **`Mc_Serial_Init2WayBalance(ctrl_t *ctrl, graph_t *graph, real_t *tpwgts)`**:
    *   Initializes balance for a 2-way partition, typically moving vertices from an overweight partition (`from=1`) to an underweight one (`to=0`) until weights are closer to `tpwgts`.
*   **`Serial_SelectQueueOneWay(idx_t ncon, real_t *npwgts, real_t *tpwgts, idx_t from, rpq_t **queues[2])`**:
    *   Selects a queue for one-way balancing, picking the constraint that is most overweight in the `from` partition.
*   **`Mc_Serial_Compute2WayPartitionParams(ctrl_t *ctrl, graph_t *graph)`**:
    *   Computes parameters (id/ed degrees, boundary) for a 2-way serial graph partition.
*   **`Serial_AreAnyVwgtsBelow(idx_t ncon, real_t alpha, real_t *vwgt1, real_t beta, real_t *vwgt2, real_t *limit)`**:
    *   Utility to check if `alpha*vwgt1 + beta*vwgt2` is below `limit` for any constraint.
*   **`ComputeSerialEdgeCut(graph_t *graph)`**: Calculates the edge cut of a serial graph.
*   **`ComputeSerialTotalV(graph_t *graph, idx_t *home)`**: Calculates total weight of vertices not in their home partition for a serial graph.

## Important Variables/Constants

*   `graph_t`: Represents the serial graph being processed.
*   `ctrl_t`: Control structure (though less MPI-dependent here, still used for workspace and parameters).
*   `where`: Partition assignment array for vertices.
*   `id`, `ed`: Internal and external degrees.
*   `bndptr`, `bndind`: Boundary vertex data structures.
*   `npwgts`: Normalized partition weights.
*   `tpwgts`: Target partition weights.
*   `rpq_t`: Priority queue type from GKlib.

## Usage Examples

```
N/A
```
These are internal ParMETIS functions. They are called by other parallel routines when a serial computation is needed on a subgraph or an assembled graph. For example:
*   `Mc_SerialKWayAdaptRefine` might be used by `Mc_Diffusion`.
*   `Mc_Serial_FM_2WayRefine` and related functions are used by `RedoMyLink`.
*   `SerialRemap` is used by `Balance_Partition`.
*   `ComputeSerialEdgeCut` and `ComputeSerialBalance` (in `stat.c`, but a version might be used here or called) are used for reporting or decision-making.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Core definitions, types, constants, GKlib utilities (memory, sorting, PQs, array ops).
    *   `macros.h`: For `BNDInsert`, `BNDDelete`, `INC_DEC`, `gk_SWAP`, `WCOREPUSH`/`POP`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: `gkMPI_Comm_rank` is called in a few places, but the result is often unused or for debugging, as these routines primarily assume serial operation on the provided `graph_t` data.
*   **Other Interactions**:
    *   These functions form the building blocks for local refinement or processing steps within larger parallel strategies.
    *   They often replicate logic found in serial METIS but are adapted for ParMETIS's data structures and multi-constraint capabilities.

```
