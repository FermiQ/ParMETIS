# libparmetis/initbalance.c

## Overview

This file implements routines for computing an initial balanced partitioning of a distributed graph, particularly in scenarios where an existing partition might be imbalanced or when a starting point for refinement is needed. The primary function `Balance_Partition` assembles the entire graph on all processors (or groups of processors) and then uses serial METIS routines or specialized internal methods (like diffusion) to compute a balanced partition. It can explore multiple strategies and pick the best one.

## Key Components

### Functions/Classes/Modules

*   **`Balance_Partition(ctrl_t *ctrl, graph_t *graph)`**:
    *   This is the main function for achieving an initial balanced partition.
    *   It first assembles the entire graph structure and weights onto each processor or within subgroups of processors using `AssembleAdaptiveGraph`. The assembled graph is referred to as `agraph`.
    *   It initializes a `home` array for `agraph` based on the original partitioning (`graph->home` or processor ID if coupled).
    *   It then splits the processors into two groups (if `ncon <= MAX_NCON_FOR_DIFFUSION` and `npes > 1`):
        *   **Group 1 (Scratch-Remap)**: Further subdivides its processors. Each subgroup recursively bisects its portion of the assembled graph using serial `METIS_PartGraphRecursive` or `METIS_PartGraphKway` to generate a candidate partition. Results are combined, and the best among these subgroups (based on cut and balance) is selected. This best partition is then potentially remapped using `SerialRemap` for better contiguity.
        *   **Group 2 (Diffusion)**: Uses diffusion-based methods (`WavefrontDiffusion` for `ncon=1` or `Mc_Diffusion` for multi-constraint) on the assembled graph to generate another candidate partition. The best result from this group is also selected and potentially remapped.
    *   If only one strategy is used (e.g., due to `ncon` or `npes`), that result is taken.
    *   The best partition between the "scratch-remap" group and the "diffusion" group is chosen based on a cost function involving cut and balance.
    *   The final chosen partition for the assembled graph is then broadcasted and scattered back to update `graph->where` on each processor for its local portion of the original distributed graph.
*   **`AssembleAdaptiveGraph(ctrl_t *ctrl, graph_t *graph)`**:
    *   Gathers the entire distributed graph (`graph`) onto every processor.
    *   Each processor sends its local graph data (CSR structure, vertex weights, vertex sizes if applicable) to all other processors using `gkMPI_Allgatherv`.
    *   Each processor then reconstructs the full graph (`agraph`) from the received data.
    *   `agraph->label` is initialized to store the original global vertex indices.
    *   Normalized vertex weights (`anvwgt`) are computed for `agraph`.

## Important Variables/Constants

*   **`agraph` (graph_t)**: The assembled graph structure containing the entire graph data, replicated on processors or subgroups.
*   **`home` (idx_t array)**: Stores the initial partition or PE assignment for vertices in `agraph`.
*   **`part` (idx_t array)**: Used to store the computed partition for `agraph`.
*   **`lwhere` (idx_t array)**: Temporary array for partition storage during parallel computations.
*   **`ctrl->ps_relation`**: Determines how `home` is initialized (`PARMETIS_PSR_COUPLED` vs. `PARMETIS_PSR_UNCOUPLED`).
*   **`ctrl->tpwgts`**: Target partition weights, used by METIS calls and remapping.
*   **`MAX_NCON_FOR_DIFFUSION`**: Constant determining if diffusion methods are attempted (usually for low numbers of constraints).
*   **`RIP_SPLIT_FACTOR`**: Factor for splitting processor groups in scratch-remap.
*   **`REDIST_WGT`**: Weight factor for redistribution cost in cost evaluation.
*   MPI Communicators: `ipcomm` (for subgroups), `srcomm` (for scratch-remap subgroups).

## Usage Examples

```
N/A
```
This is an internal ParMETIS function. `Balance_Partition` is typically called by higher-level routines like `AdaptiveRepart` when the initial partition is imbalanced or needs to be established before refinement.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides `ctrl_t`, `graph_t` definitions, constants, MPI wrappers, memory utilities, etc.
    *   `graph.c`: `AssembleAdaptiveGraph` uses `CreateGraph`, `imalloc`, `rmalloc`, `icopy`, `iincset`, `MAKECSR`. `Balance_Partition` uses `AssembleAdaptiveGraph`, `FreeGraph`.
    *   `diffutil.c`: `WavefrontDiffusion`, `Mc_Diffusion` are called if the diffusion strategy is chosen.
    *   `serial.c`: `SerialRemap`, `KeepPart`, `ComputeSerialEdgeCut`, `ComputeSerialBalance`.
    *   `ctrl.c`: For `AllocateWSpace`, `AllocateRefinementWorkSpace`, `FreeCtrl` (for temporary ctrl).
    *   `metis.h` (external METIS library): `METIS_SetDefaultOptions`, `METIS_PartGraphRecursive`, `METIS_PartGraphKway`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Used extensively for communication (`gkMPI_Allgatherv`, `gkMPI_Allreduce`, `gkMPI_Send`, `gkMPI_Recv`, `gkMPI_Bcast`, `gkMPI_Comm_split`, `gkMPI_Comm_rank`, `gkMPI_Comm_size`, `gkMPI_Comm_free`).
    *   `METIS`: Serial METIS library is used by the "scratch-remap" strategy to partition the assembled graph.
*   **Other Interactions**:
    *   The choice between scratch-remap and diffusion strategies, and the selection of the final best partition, depend on the number of constraints and the resulting quality (cut, balance, cost).
    *   Modifies `graph->where` with the new balanced partition.

```
