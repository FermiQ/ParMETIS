# libparmetis/mdiffusion.c

## Overview

This file implements multi-constraint diffusion (Mc_Diffusion) and related utilities for load balancing in ParMETIS. Diffusion algorithms attempt to balance workload by modeling the flow of work between partitions, often by solving a system of linear equations. This file includes functions to set up the problem (e.g., graph of partition connectivity, load vectors), solve it using Conjugate Gradient, and then apply the solution to move vertices or adjust partition assignments. It also contains a specific routine `BalanceMyLink` (though a more prominent version exists in `balancemylink.c`) and `RedoMyLink` which seem to be local refinement/balancing operations between two specific partitions.

## Key Components

### Functions/Classes/Modules

*   **`Mc_Diffusion(ctrl_t *ctrl, graph_t *graph, idx_t *vtxdist, idx_t *where, idx_t *home, idx_t npasses)`**:
    *   The main multi-constraint diffusion function.
    *   It iterates for `npasses` or until balance is achieved.
    *   In each pass:
        1.  Constructs a "connectivity graph" of partitions (`matrix`) using `SetUpConnectGraph`.
        2.  For each constraint `h`:
            *   Computes current load imbalance (`ComputeLoad`).
            *   Solves a linear system `Ax = b` using `ConjGrad2`, where `A` is the partition connectivity matrix and `b` is the load imbalance. The solution `x` represents "potentials" or "pressures".
            *   Computes `transfer` amounts along edges of the partition graph based on `x` (`ComputeTransferVector`).
        3.  Identifies independent sets of inter-partition links (edges in `matrix`) to process in parallel using `CSR_Match_SHEM`.
        4.  For each matched pair of partitions (`me`, `you`) assigned to the current PE:
            *   Extracts the subgraph `egraph` consisting of vertices in `me` or `you` (`ExtractGraph`).
            *   Calls `RedoMyLink` and/or `BalanceMyLink` (a local 2-way refinement/balancing function) on `egraph` to adjust vertex assignments between `me` and `you` based on the computed `diff_flows` (derived from `transfer`).
            *   The choice between `RedoMyLink` and `BalanceMyLink` results might depend on balance and cost.
        5.  Communicates updated `where` information for moved vertices.
        6.  Optionally performs serial k-way refinement (`Mc_SerialKWayAdaptRefine`) on the entire graph (assembled locally, though this part is not fully shown in `Mc_Diffusion` but implied by usage pattern in other files).
        7.  Checks for convergence based on overall balance (`lbavg`) or if no swaps occurred.
*   **`SetUpConnectGraph(graph_t *graph, matrix_t *matrix, idx_t *workspace)`**:
    *   (Duplicate of the one in `diffutil.c`) Builds a coarse matrix representing connectivity between current partitions in `graph->where`.
*   **`Mc_ComputeMoveStatistics(ctrl_t *ctrl, graph_t *graph, idx_t *nmoved, idx_t *maxin, idx_t *maxout)`**:
    *   (Duplicate of the one in `diffutil.c`) Computes statistics about vertex movements.
*   **`Mc_ComputeSerialTotalV(graph_t *graph, idx_t *home)`**:
    *   (Duplicate of the one in `diffutil.c`) Computes total weight of vertices not in their home partition.
*   **`ComputeLoad(graph_t *graph, idx_t nparts, real_t *load, real_t *tpwgts, idx_t index)`**:
    *   (Duplicate of the one in `diffutil.c`) Computes load imbalance for a specific constraint.
*   **`ConjGrad2(matrix_t *A, real_t *b, real_t *x, real_t tol, real_t *workspace)`**:
    *   (Duplicate of the one in `diffutil.c`) Conjugate Gradient solver.
*   **`mvMult2(matrix_t *A, real_t *v, real_t *w)`**:
    *   (Duplicate of the one in `diffutil.c`) Matrix-vector multiplication.
*   **`ComputeTransferVector(idx_t ncon, matrix_t *matrix, real_t *solution, real_t *transfer, idx_t index)`**:
    *   (Duplicate of the one in `diffutil.c`) Computes transfer amounts from solution potentials.
*   **`ExtractGraph(ctrl_t *ctrl, graph_t *graph, idx_t *indicator, idx_t *map, idx_t *rmap)`**:
    *   Extracts a subgraph containing only vertices marked by the `indicator` array.
    *   `map` stores the original indices of the extracted vertices, `rmap` maps original indices to extracted indices.
    *   Used in `Mc_Diffusion` to create a temporary graph for `BalanceMyLink`/`RedoMyLink`.

## Important Variables/Constants

*   **`matrix_t matrix`**: Represents the graph of connections between partitions.
*   **`transfer`**: Array storing flow amounts computed for edges in `matrix`.
*   **`solution`, `load`**: Vectors used in the CG solver for potentials and load imbalances.
*   **`egraph`**: Temporary graph structure for vertices in two partitions being balanced.
*   **`diff_flows`, `sr_flows`**: Flow values passed to `BalanceMyLink` and `RedoMyLink`.
*   `N_MOC_GD_PASSES` (from `defs.h`): Likely related to number of global diffusion passes, though `npasses` is passed to `Mc_Diffusion`.
*   `NGD_PASSES` (from `defs.h`): Maximum iterations for the inner loop of `Mc_Diffusion`.

## Usage Examples

```
N/A
```
These are internal ParMETIS functions. `Mc_Diffusion` is a load-balancing algorithm that might be called during adaptive repartitioning (`ametis.c`) or as part of an initial balancing phase (`initbalance.c`).

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Core definitions, types, constants.
    *   `diffutil.c`: This file duplicates several functions from `diffutil.c`. Ideally, these would be shared.
    *   `csrmatch.c`: `CSR_Match_SHEM` is used to find independent sets of inter-partition links.
    *   `balancemylink.c`: Contains `BalanceMyLink`.
    *   `redomylink.c`: Contains `RedoMyLink`.
    *   `kwayrefine.c` (indirectly): `Mc_SerialKWayAdaptRefine` is called.
    *   `graph.c`: For `CreateGraph`, `ExtractGraph` uses `CreateGraph`, `MAKECSR`.
    *   `GKlib` utilities: For memory allocation, array operations.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Used for communication (`gkMPI_Allgatherv`, `gkMPI_Allreduce`, `GlobalSESum`).
*   **Other Interactions**:
    *   `Mc_Diffusion` modifies the `graph->where` array to balance loads.
    *   The effectiveness depends on the CG solver's convergence and the heuristics in `BalanceMyLink`/`RedoMyLink`.
    *   The duplication of several utility functions also found in `diffutil.c` is notable and could be a point of refactoring.

```
