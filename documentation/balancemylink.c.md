# libparmetis/balancemylink.c

## Overview

This file implements a specialized edge-based Fiduccia-Mattheyses (FM) refinement algorithm. The primary function, `BalanceMyLink`, aims to balance the "flow" (representing workload or data imbalance) between two specific, adjacent partitions, denoted as `me` and `you`. It iteratively moves vertices between these two partitions to reduce the specified flow imbalance while trying to minimize edge cut and considering data redistribution costs. This routine is likely used as a low-level component within more complex load balancing or adaptive refinement schemes in ParMETIS.

## Key Components

### Functions/Classes/Modules

*   **`BalanceMyLink(ctrl_t *ctrl, graph_t *graph, idx_t *home, idx_t me, idx_t you, real_t *flows, real_t maxdiff, real_t *diff_cost, real_t *diff_lbavg, real_t avgvwgt)`**:
    *   This is the sole function in this file. It performs a 2-way FM-based refinement/balancing pass between partitions `me` and `you`.
    *   **Parameters**:
        *   `ctrl`: Control structure containing global parameters like `ipc_factor` and `redist_factor`.
        *   `graph`: Graph structure containing local graph information (vertices, edges, weights, current partition in `graph->where`).
        *   `home`: An array indicating the original "home" partition for each vertex. This is used to calculate redistribution costs/gains.
        *   `me`, `you`: The identifiers of the two partitions between which balancing is performed. Vertices considered must belong to either `me` or `you`.
        *   `flows`: An array representing the current imbalance (or "flow" of weights/load) for each constraint between partition `me` and `you`. Positive values might mean `me` needs to send to `you`, negative means `you` needs to send to `me`. The function aims to reduce these flow values.
        *   `maxdiff`: A threshold related to the desired level of balance or maximum allowed difference.
        *   `diff_cost` (output): The total cost (considering edge cut and redistribution) after the balancing attempt.
        *   `diff_lbavg` (output): The average load imbalance between `me` and `you` after balancing.
        *   `avgvwgt`: Average vertex weight, potentially used in heuristics for selecting queues or vertices.
    *   **Functionality**:
        1.  Initializes local data structures, including workspace for vertex degrees, mappings, and priority queues.
        2.  Calculates initial total partition weights (`pwgts`) for `me` and `you`, and then target partition weights (`tpwgts`) based on the initial weights and the input `flows`.
        3.  Categorizes vertices by a hash of their weights (`Mc_HashVwgts`) into multiple priority queues (`queues`). Separate sets of queues are maintained for vertices in partition `me` and partition `you`.
        4.  Computes initial internal (`id`) and external (`ed`) degrees for each vertex with respect to partitions `me` and `you`.
        5.  Performs a number of FM passes (`N_MOC_BAL_PASSES`):
            *   Resets and populates priority queues. For each vertex, calculates a gain based on `ipc_factor` (reduction in edge cut) and `redist_factor` (cost/benefit of moving from/to its `home` partition).
            *   Tracks the `bestflow` achieved (minimum absolute flow value).
            *   Iteratively selects the most promising queue and vertex to move using `Mc_DynamicSelectQueue`, which considers the current `flows` and `avgvwgt`.
            *   Moves the selected vertex (`vtx`) from its current partition (`from`) to the other (`to`). Updates `where[vtx]`.
            *   Updates the `flows` array based on the moved vertex's weights.
            *   If the new `flows` represent a better balance (`ftmp < bestflow`), resets a list of `changes`. Otherwise, adds the moved vertex to `changes`.
            *   Updates the `id` and `ed` of the moved vertex and its neighbors, and updates their gains in the priority queues.
            *   After iterating (e.g., up to `nvtxs/2` potential moves), it rolls back moves stored in `changes` to revert to the state that achieved `bestflow`.
            *   If no moves were made in a pass, it breaks early.
        6.  After all passes, calculates the final load balance (`diff_lbavg`) between `me` and `you` based on the final `where` assignments and `tpwgts`.
        7.  Calculates the final cost (`diff_cost`) based on the resulting edge cut between `me` and `you` and the total `vsize` of vertices not in their `home` partition.
        8.  Frees allocated workspace, including priority queues.
        9.  Returns the total number of successful swaps (`nswaps`).

## Important Variables/Constants

*   **Input/Output Parameters**:
    *   `me`, `you`: The two partitions involved in balancing.
    *   `flows`: Array (input/output) representing weight imbalance for `ncon` constraints; the function aims to reduce these values.
    *   `home`: Array indicating the "ideal" or original partition for each vertex.
    *   `diff_cost`, `diff_lbavg`: Output values for cost and load balance.
*   **Graph Data (from `graph_t`)**:
    *   `nvtxs`, `ncon`, `xadj`, `adjncy`, `adjwgt`, `nvwgt`, `vsize`, `where`.
*   **Control Parameters (from `ctrl_t`)**:
    *   `ipc_factor`: Weight for inter-processor communication (edge cut) in gain calculations.
    *   `redist_factor`: Weight for data redistribution cost in gain calculations.
*   **FM Algorithm Data Structures**:
    *   `queues (rpq_t **)`: Array of pointers to priority queues, used to select vertices with high gain.
    *   `hval`: Array storing a hash value for each vertex (based on its `nvwgt`), used for queue assignment.
    *   `id`, `ed`: Arrays storing internal and external weighted degrees of vertices with respect to partitions `me` and `you`.
    *   `myqueue`: Array indicating if a vertex is currently in a queue and for which partition.
    *   `map`, `rmap`: Mapping arrays for efficient access to vertices within the queues.
    *   `changes`: Array to log vertex moves, enabling rollback to the best state.
*   **Partition Weight Tracking**:
    *   `pwgts`: Initial weights of partitions `me` and `you`.
    *   `tpwgts`: Target weights for partitions `me` and `you`, derived from `pwgts` and `flows`.
*   **Constants**:
    *   `N_MOC_BAL_PASSES`: Defines the number of passes the FM algorithm will perform.
    *   `PE 0`: Defined but unused in the logic.

## Usage Examples

```
N/A
```
This function is a specialized internal routine within ParMETIS. It is not intended for direct use by end-users. It's likely called by higher-level functions that manage overall load balancing or adaptive partitioning, which might identify pairs of partitions (`me`, `you`) and specific `flows` between them that need adjustment.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Directly included. This header provides:
        *   Core data structures: `ctrl_t`, `graph_t`.
        *   Priority Queue (`rpq_t`) API: `rpqCreate`, `rpqInsert`, `rpqGetTop`, `rpqUpdate`, `rpqReset`, `rpqDestroy`.
        *   Memory management: `iwspacemalloc`, `rwspacemalloc`, `wspacemalloc`, `rset`, `iset`, `WCOREPUSH`/`WCOREPOP`.
        *   Helper functions: `Mc_HashVwgts` (for categorizing vertices by weight for queueing), `Mc_DynamicSelectQueue` (for selecting which queue/vertex to process next), `ravg` (for averaging).
        *   Macros: `ASSERT`, `gk_SWAP`, `INC_DEC`.
    *   It is expected to be called by other ParMETIS functions that orchestrate larger-scale balancing or refinement, which would determine the `me`, `you`, and `flows` parameters.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: `gkMPI_Comm_rank(MPI_COMM_WORLD, &mype)` is called at the beginning, but the returned `mype` is not subsequently used in the core logic. The function primarily operates on data assumed to be locally available for the specified `me` and `you` partitions, implying that the calling context handles the necessary data distribution or setup.
*   **Other Interactions**:
    *   The function modifies the `graph->where` array for vertices that are moved between partitions `me` and `you`.
    *   The input `flows` array is modified in place to reflect the changes due to vertex moves.
    *   The `diff_cost` and `diff_lbavg` variables are updated to provide feedback to the caller on the outcome of the balancing attempt.
    *   The `home` array is read-only and influences gain calculations related to redistribution.

```
