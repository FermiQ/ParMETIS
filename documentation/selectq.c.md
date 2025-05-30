# libparmetis/selectq.c

## Overview

This file provides utility functions for selecting which priority queue to draw a vertex from during certain multi-constraint refinement or balancing operations, specifically within diffusion-based algorithms (`mdiffusion.c`). It also includes helper functions to compute hash values based on vertex weights, which are used to distribute vertices among multiple priority queues. The queue selection logic aims to prioritize moves that best alleviate imbalance across multiple constraints.

## Key Components

### Functions/Classes/Modules

*   **`Mc_DynamicSelectQueue(ctrl_t *ctrl, idx_t nqueues, idx_t ncon, idx_t subdomain1, idx_t subdomain2, idx_t *currentq, real_t *flows, idx_t *from, idx_t *qnum, idx_t minval, real_t avgvwgt, real_t maxdiff)`**:
    *   Dynamically selects a queue (`*qnum`) and a source partition (`*from`) from which to pick a vertex for a potential move. This is used in multi-constraint scenarios where vertices are categorized into `nqueues` (based on weight hashes) and there are `ncon` constraints.
    *   **Parameters**:
        *   `nqueues`: Number of queues per partition side.
        *   `ncon`: Number of constraints.
        *   `subdomain1`, `subdomain2`: The two subdomains/partitions involved in the potential move.
        *   `currentq`: An array indicating the number of elements currently in each actual priority queue.
        *   `flows`: An array of size `ncon` indicating the current imbalance for each constraint between `subdomain1` and `subdomain2`.
        *   `from` (output): The selected source subdomain (either `subdomain1` or `subdomain2`). If initially -1, it's determined by the largest flow.
        *   `qnum` (output): The index of the selected queue (0 to `nqueues-1`).
        *   `minval`: Minimum hash value, used to adjust `qnum`.
        *   `avgvwgt`: Average vertex weight, used as a threshold with `MOC_GD_GRANULARITY_FACTOR`.
        *   `maxdiff`: Maximum difference between sorted flow components to consider them "close".
    *   **Functionality**:
        1.  If `*from` is -1, determines the initial `*from` partition based on the constraint with the largest absolute `flows` value. If `flows[c] > avgvwgt_thresh`, `*from = subdomain1`; if `flows[c] < -avgvwgt_thresh`, `*from = subdomain2`.
        2.  Sorts the `flows` (multiplied by a `sign` depending on `*from`) to determine the order of importance of constraints.
        3.  Identifies "don't care" constraints: if flow values for adjacent constraints in the sorted list are close (difference `< maxdiff * MC_FLOW_BALANCE_THRESHOLD`), they are grouped.
        4.  Based on `ncon` (hardcoded for 2, 3, 4 constraints) and the "don't care" grouping, it generates a list of candidate permutations (`perm`) of constraint indices. These permutations represent different orderings of preference for satisfying constraints.
        5.  Iterates through these permutations. For each permutation, it forms a hash value representing a vertex type that would be ideal to move (using `Mc_HashVRank`).
        6.  If a queue corresponding to this hash value (`hash - minval + index`) has elements (`currentq[q_idx] > 0`), that queue `*qnum` is selected, and the function returns.
*   **`Mc_HashVwgts(ctrl_t *ctrl, idx_t ncon, real_t *nvwgt)`**:
    *   Computes a hash value for a vertex based on its normalized weights (`nvwgt`) across `ncon` constraints.
    *   It first sorts the constraints by the vertex's weight in them.
    *   Then, it calculates a unique integer based on the ranks of these sorted weights. This hash value is used to assign vertices to one of `nqueues` in refinement algorithms like `BalanceMyLink`.
*   **`Mc_HashVRank(idx_t ncon, idx_t *vwgt)`**:
    *   Computes a hash value based on an already determined ranking of constraints for a vertex.
    *   `vwgt` here is expected to be an array of ranks (0 to `ncon-1`).
    *   Calculates `retval = sum(vwgt[ncon-1-i] * (i+1)!)`. This provides a unique integer for each permutation of ranks.

## Important Variables/Constants

*   **`flows`**: Array of imbalances for each constraint. Drives the selection of which constraint to prioritize.
*   **`perm` (in `Mc_DynamicSelectQueue`)**: A 2D array holding pre-defined permutations of constraint indices for `ncon` = 2, 3, 4. These permutations guide the search for a suitable queue.
*   **`MOC_GD_GRANULARITY_FACTOR`**: Multiplier for `avgvwgt` to set a threshold for flow significance.
*   **`MC_FLOW_BALANCE_THRESHOLD`**: Factor for `maxdiff` to determine if flows are "close enough" to be grouped as "don't cares".

## Usage Examples

```
N/A
```
These are internal helper functions for ParMETIS, specifically for multi-constraint refinement and balancing algorithms.
*   `Mc_DynamicSelectQueue` would be called within a loop in a routine like `BalanceMyLink` (in `balancemylink.c`) or the diffusion code to decide which type of vertex (from which queue) to consider moving next.
*   `Mc_HashVwgts` is used to assign vertices to specific queues based on their weight distributions when initializing those queues.
*   `Mc_HashVRank` is used by `Mc_DynamicSelectQueue` to generate target hash values based on prioritized constraint rankings.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides `ctrl_t`, `idx_t`, `real_t`, `rkv_t` types, MPI wrappers (`gkMPI_Comm_rank`), memory utilities (`iwspacemalloc`, `rkvwspacemalloc`, `WCOREPUSH`/`POP`), sorting (`rkvsorti`).
    *   `mdiffusion.c` and `balancemylink.c` (or similar refinement/balancing routines) are the primary callers of these functions.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: `gkMPI_Comm_rank` is called, but its result `mype` is not used in the logic of these specific functions.
*   **Other Interactions**:
    *   The hardcoded permutations in `Mc_DynamicSelectQueue` for `ncon` up to 4 limit its direct applicability for a higher number of constraints without extension.
    *   The hashing functions are designed to group vertices with similar weight distributions or rankings, allowing refinement heuristics to target specific vertex types.

```
