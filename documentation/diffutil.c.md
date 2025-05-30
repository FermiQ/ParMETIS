# libparmetis/diffutil.c

## Overview

This file contains utility functions primarily supporting diffusion-based load balancing and adaptive refinement schemes within ParMETIS. This includes setting up a "connectivity graph" or matrix representing connections between partitions, computing statistics about vertex movements, calculating loads, and a Conjugate Gradient (CG) solver likely used for solving linear systems that arise in diffusion algorithms.

## Key Components

### Functions/Classes/Modules

*   **`SetUpConnectGraph(graph_t *graph, matrix_t *matrix, idx_t *workspace)`**:
    *   Constructs a coarse-grained matrix (`matrix`) representing the connectivity between partitions based on the fine-grained `graph`.
    *   The rows of the `matrix` correspond to partitions. `matrix->values[rowptr[ii]]` stores the number of external connections for partition `ii`, and `matrix->values[k] = -1.0` for off-diagonal entries `colind[k]` indicate a connection to another partition.
    *   Uses a `workspace` for permutations and marking.
*   **`Mc_ComputeMoveStatistics(ctrl_t *ctrl, graph_t *graph, idx_t *nmoved, idx_t *maxin, idx_t *maxout)`**:
    *   Computes statistics about vertex movements for multi-constraint adaptive refinement.
    *   Calculates:
        *   `nmoved`: Total amount of vertex weight/size that moved from its original partition (`graph->home` or `ctrl->mype` for coupled mode).
        *   `maxin`: Maximum total weight/size of vertices that moved into any single partition.
        *   `maxout`: Maximum total weight/size of vertices that moved out of any single partition.
    *   Uses MPI_Allreduce to get global sums.
*   **`Mc_ComputeSerialTotalV(graph_t *graph, idx_t *home)`**:
    *   Computes the total "volume" or weight of vertices that are not in their specified `home` partition in a serial graph context (or for the local portion of a distributed graph).
    *   Uses `graph->vsize` if available, otherwise the first component of `graph->vwgt`.
*   **`ComputeLoad(graph_t *graph, idx_t nparts, real_t *load, real_t *tpwgts, idx_t index)`**:
    *   Computes the current load for each of `nparts` partitions for a specific constraint `index`.
    *   The load is calculated as `actual_weight[p] - target_weight[p]`.
    *   The sum of `actual_weight` is asserted to be close to 1.0 (implying normalized weights).
*   **`ConjGrad2(matrix_t *A, real_t *b, real_t *x, real_t tol, real_t *workspace)`**:
    *   Implements a Conjugate Gradient (CG) solver to find `x` in `Ax = b`.
    *   `A` is a matrix (likely the connectivity matrix from `SetUpConnectGraph`).
    *   `b` is the right-hand side vector (e.g., load imbalances).
    *   `x` is the solution vector (e.g., desired flow).
    *   `tol` is the error tolerance.
    *   `workspace` is used for temporary vectors `p, r, q, z, M` (preconditioner).
    *   The preconditioner `M` is the inverse of the diagonal of `A`.
*   **`mvMult2(matrix_t *A, real_t *v, real_t *w)`**:
    *   Performs a matrix-vector multiplication `w = A*v` for the `matrix_t` structure.
*   **`ComputeTransferVector(idx_t ncon, matrix_t *matrix, real_t *solution, real_t *transfer, idx_t index)`**:
    *   Based on a `solution` vector (likely from `ConjGrad2`, representing potentials or pressures), computes a `transfer` vector.
    *   For each edge `(j, colind[k])` in the `matrix`, if `solution[j] > solution[colind[k]]`, then `transfer[k*ncon+index]` is set to their difference, otherwise 0. This quantifies potential flow along matrix edges.

## Important Variables/Constants

*   **`matrix_t`**: Structure representing a sparse matrix, typically the connectivity between partitions. Key members: `nrows`, `rowptr`, `colind`, `values`, `nnzs`.
*   **`graph_t`**: Structure for the fine-grained graph. Key members used: `nvtxs`, `xadj`, `adjncy`, `where`, `vwgt`, `vsize`, `ncon`, `home`.
*   **`ctrl_t`**: Control structure, used for `ps_relation`, `mype`, `nparts`, and MPI communicator.

## Usage Examples

```
N/A
```
These functions are internal utilities for ParMETIS, primarily supporting diffusion-based load balancing algorithms.
*   `SetUpConnectGraph` would be called to build a coarse representation of inter-partition connectivity.
*   `ComputeLoad` would calculate imbalances.
*   `ConjGrad2` would solve for corrective flows based on the connectivity and imbalances.
*   `ComputeTransferVector` would translate the solution of the linear system into concrete transfer amounts along graph edges.
*   `Mc_ComputeMoveStatistics` and `Mc_ComputeSerialTotalV` help in analyzing the results of partitioning or refinement.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides `graph_t`, `matrix_t`, `ctrl_t` definitions, `idx_t`, `real_t` types, MPI wrappers (`gkMPI_Allreduce`), memory utilities (`iset`, `ismalloc`, `gk_free`), CSR utilities (`MAKECSR`), math utilities (`fabs`, `rsum`, `rnorm2`, `rdot`, `rcopy`, `raxpy`, `isum`, `imax`), and `ASSERT`.
    *   These utilities are likely called by higher-level diffusion algorithms (e.g., `mdiffusion.c`) or adaptive refinement schedulers.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Used in `Mc_ComputeMoveStatistics` for `gkMPI_Allreduce`.
*   **Other Interactions**:
    *   The functions operate on and modify data within `graph_t` and `matrix_t` structures.
    *   The CG solver assumes a symmetric positive definite (or semi-definite) matrix, which is typical for diffusion problems.

```
