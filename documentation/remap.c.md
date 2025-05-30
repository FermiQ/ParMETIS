# libparmetis/remap.c

## Overview

This file contains functions related to remapping partitions to processors, primarily to minimize data redistribution costs. When a k-way partition is computed, the initial assignment of partition numbers (0 to k-1) to physical processor ranks (0 to P-1, where k=P) might not be optimal in terms of how much data needs to move if the graph was already distributed. These functions try to find a permutation of this assignment to reduce total data movement.

## Key Components

### Functions/Classes/Modules

*   **`ParallelReMapGraph(ctrl_t *ctrl, graph_t *graph)`**:
    *   This function orchestrates the remapping process if the number of processors (`ctrl->npes`) equals the number of partitions (`ctrl->nparts`).
    *   It calculates `lpwgts`: the sum of `vsize` (or 1 if `vsize` is NULL) of local vertices currently assigned to each partition.
    *   Calls `ParallelTotalVReMap` to compute the optimal mapping (`map` array).
    *   Updates `graph->where` by applying the computed `map` to existing partition assignments (`where[i] = map[where[i]]`).
*   **`ParallelTotalVReMap(ctrl_t *ctrl, idx_t *lpwgts, idx_t *map, idx_t npasses, idx_t ncon)`**:
    *   Computes an optimal assignment of partitions to processors to minimize total data movement (total volume).
    *   It assumes `nparts == npes`.
    *   **Algorithm**:
        1.  Initializes `map` (partition-to-PE mapping) to -1 (unassigned) and `rowmap` (PE-to-partition mapping) to -1.
        2.  Iterates for `npasses` or until all partitions are mapped:
            *   Each processor `mype` identifies its local partition `maxipwgt` that currently holds the most local data (from `mylpwgts`, a copy of `lpwgts`).
            *   It proposes to map this local partition `maxipwgt` to itself (`mype`), sending a pair `(-mylpwgts[maxipwgt], mype*nparts+maxipwgt)` to all PEs. The negative weight is for sorting to pick heaviest.
            *   All PEs gather these proposals (`gkMPI_Allgather` into `recv` array).
            *   Proposals are sorted by weight (heaviest first).
            *   Iterates through sorted proposals: If partition `j` (proposed by PE `k`) is unmapped (`map[j] == -1`) and PE `k` is unassigned (`rowmap[k] == -1`), and their target partition weights (`ctrl->tpwgts`) are "similar" (`SimilarTpwgts`), then assign `map[j] = k` and `rowmap[k] = j`. Mark `mylpwgts[j] = 0` on the PE that originally held it to prevent re-proposing.
        3.  If any partitions remain unmapped after the passes (due to `tpwgts` dissimilarity perhaps), they are assigned greedily.
        4.  Calculates the total data saved by this remapping. If no savings (or negative savings), it reverts `map` to the identity mapping (`map[i] = i`).
*   **`SimilarTpwgts(real_t *tpwgts, idx_t ncon, idx_t s1, idx_t s2)`**:
    *   Checks if the target partition weights (`tpwgts`) for two partitions (or PEs, as `nparts==npes`) `s1` and `s2` are similar enough (within `SMALLFLOAT` for all constraints).
    *   Returns 1 if similar, 0 otherwise. This is used in `ParallelTotalVReMap` to ensure that a partition `j` is only mapped to a processor `k` if their target workloads are compatible.

## Important Variables/Constants

*   **`map` (in `ParallelTotalVReMap`, output to `ParallelReMapGraph`)**: An array of size `nparts`. `map[i] = j` means original partition `i` should be remapped to new partition ID `j` (which corresponds to PE `j`).
*   **`lpwgts` (in `ParallelTotalVReMap`)**: Array storing the amount of data each PE has for each partition.
*   **`rowmap` (in `ParallelTotalVReMap`)**: Temporary array, `rowmap[k] = j` means PE `k` is now assigned partition `j`.
*   **`ctrl->tpwgts`**: Target partition weights, used by `SimilarTpwgts`.
*   **`NREMAP_PASSES`**: Constant from `defs.h`, number of passes for the greedy mapping heuristic.
*   **`SMALLFLOAT`**: Constant from `defs.h`, used in `SimilarTpwgts` for floating point comparisons.

## Usage Examples

```
N/A
```
These are internal ParMETIS functions. `ParallelReMapGraph` is typically called after a k-way partitioning (e.g., in `ParMETIS_V3_PartKway` or `ParMETIS_V3_AdaptiveRepart`) when `nparts == npes`, to optimize the assignment of the computed partitions to the physical processors.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Core definitions, types, constants.
    *   GKlib utilities: `iset`, `icopy`, `iargmax`, `ikvsorti`, `GlobalSESum`.
    *   `graph.c` (specifically `graph->vsize`, `graph->where`).
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Used for `gkMPI_Allgather`, `gkMPI_Scan` (implicitly if used by `GlobalSESum`).
*   **Other Interactions**:
    *   Modifies `graph->where` by applying the computed `map`.
    *   The remapping is only effective if `nparts == npes`.
    *   The `SimilarTpwgts` check prevents remapping partitions in a way that would violate target weight compatibility between the original partition slot and the target processor's slot, which is important for heterogeneous target weights.

```
