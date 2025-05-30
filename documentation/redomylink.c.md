# libparmetis/redomylink.c

## Overview

This file contains the `RedoMyLink` function, which appears to be a specialized 2-way refinement/balancing routine. It's similar in purpose to `BalanceMyLink` (found in `balancemylink.c` and also duplicated in `mdiffusion.c`), aiming to improve the partition between two specific subdomains (`me` and `you`) based on `flows` (imbalance). It tries multiple random initial bisections and refines them using serial METIS-like FM refinement and balancing steps, ultimately selecting the best outcome based on cost and load balance.

## Key Components

### Functions/Classes/Modules

*   **`RedoMyLink(ctrl_t *ctrl, graph_t *graph, idx_t *home, idx_t me, idx_t you, real_t *flows, real_t *sr_cost, real_t *sr_lbavg)`**:
    *   Performs a 2-way refinement/balancing between specified partitions `me` and `you`.
    *   **Parameters**:
        *   `ctrl`: Control structure.
        *   `graph`: The graph to operate on. Assumed to contain only vertices belonging to `me` or `you`.
        *   `home`: Original partition assignments for vertices in `graph`.
        *   `me`, `you`: The two partitions being balanced.
        *   `flows`: Input array representing the imbalance for each constraint that needs to be corrected.
        *   `sr_cost` (output): The cost (cut + redistribution) of the best partition found.
        *   `sr_lbavg` (output): The load balance average of the best partition found.
    *   **Functionality**:
        1.  Initializes workspace and data structures, including temporary `where` arrays (`costwhere`, `lbwhere`) to store the best partitions found according to cost and load balance, respectively.
        2.  Calculates target partition weights (`tpwgts`) for the 2-way problem based on current actual weights (`pwgts`) and the input `flows`.
        3.  Performs `N_MOC_REDO_PASSES` iterations:
            *   In each pass, it creates a random initial bisection of the input `graph` (by assigning one randomly chosen vertex to partition 0, others to 1, which are then mapped to `me` and `you`).
            *   Calls `Mc_Serial_Compute2WayPartitionParams` to set up parameters for the 2-way problem.
            *   Calls `Mc_Serial_Init2WayBalance` to perform initial balancing towards `tpwgts`.
            *   Calls `Mc_Serial_FM_2WayRefine` (an FM-like refinement) multiple times.
            *   Calls `Mc_Serial_Balance2Way` to further improve balance.
            *   Calculates the resulting load balance (`lbavg`) and cost (`mycost` = `ipc_factor * cut + redist_factor * total_moved_volume`).
            *   Updates `bestcost`, `other_lbavg`, `costwhere` if `mycost` is better.
            *   Updates `best_lbavg`, `othercost`, `lbwhere` if `lbavg` is better.
        4.  After all passes, it selects between `costwhere` and `lbwhere`. If the balance achieved by `costwhere` (`other_lbavg`) is very good (e.g., <= 0.05), it chooses `costwhere`. Otherwise, it chooses `lbwhere` (which prioritized balance).
        5.  Copies the selected partition into `graph->where`.
        6.  Sets output parameters `*sr_cost` and `*sr_lbavg`.

## Important Variables/Constants

*   **`me`, `you`**: The two partitions involved.
*   **`flows`**: Input array indicating the imbalance to be corrected.
*   **`home`**: Array of original partition assignments.
*   **`costwhere`, `lbwhere`**: Temporary arrays to store the partition vectors that yielded the best cost and best load balance, respectively.
*   **`tpwgts`**: Target partition weights for the 2-way problem.
*   **`ipc_factor`, `redist_factor`**: From `ctrl`, used in cost calculation.
*   **`N_MOC_REDO_PASSES`**: Constant from `defs.h` defining the number of refinement attempts.

## Usage Examples

```
N/A
```
This is an internal ParMETIS function. It's likely called by higher-level diffusion or refinement algorithms (e.g., `Mc_Diffusion`) when a specific link (pair of partitions) needs its balance and cut characteristics improved. The calling function would typically extract the subgraph relevant to partitions `me` and `you` before calling `RedoMyLink`.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Core definitions, types, constants.
    *   `serial.c`: This function heavily relies on serial 2-way refinement and balancing routines:
        *   `Mc_Serial_Compute2WayPartitionParams`
        *   `Mc_Serial_Init2WayBalance`
        *   `Mc_Serial_FM_2WayRefine`
        *   `Mc_Serial_Balance2Way`
    *   `util.c`: For `RandomPermute`.
    *   GKlib utilities: For memory allocation (`iwspacemalloc`, `rwspacemalloc`, `rset`, `iset`, `icopy`) and array operations (`isum`, `ravg`).
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Only `gkMPI_Comm_rank` is called, and its result `mype` is not used in the logic, suggesting this function operates on data assumed to be entirely local (e.g., on an extracted subgraph).
*   **Other Interactions**:
    *   Modifies `graph->where` with the refined 2-way partition.
    *   The function attempts multiple random starts to avoid getting stuck in poor local minima.
    *   The choice between the best-cost and best-balance solutions introduces a heuristic to favor balance unless the cost-optimized solution is already very well-balanced.

```
