# libparmetis/initmsection.c

## Overview

This file implements the initial k-way multisection algorithm for ParMETIS. It's designed to find vertex separators that divide a graph (or parts of it) into multiple sections simultaneously. The process involves assembling the graph, splitting processors into groups, having each group compute a bisection (separator) of its assigned part of the graph using serial METIS, and then selecting the best separator among the groups. The final partitioning, including separators, is encoded in the `graph->where` array.

## Key Components

### Functions/Classes/Modules

*   **`InitMultisection(ctrl_t *ctrl, graph_t *graph)`**:
    *   The main entry point for computing an initial multisection.
    *   It first assembles the graph from all processors onto each processor within defined subgroups using `AssembleMultisectedGraph`. The assembled graph is `agraph`.
    *   Processors are split into `ctrl->nparts / 2` groups using `MPI_Comm_split`. Each group is responsible for finding a 2-way separator for a distinct portion of the graph.
    *   `mypart = ctrl->mype % (ctrl->nparts / 2)` determines which part of the graph this PE group focuses on.
    *   `KeepPart` is called to isolate the relevant subgraph within `agraph` for each processor group.
    *   Serial METIS (`METIS_ComputeVertexSeparator`) is called by each group on its subgraph to find a 2-way separator (partitions 0, 1, and separator 2).
    *   The local separator result (0, 1, 2) is then re-labeled globally: vertices in partition 0 of subproblem `mypart` become `2*mypart`, partition 1 becomes `2*mypart + 1`, and separator vertices become `ctrl->nparts + 2*mypart`.
    *   Within each group, the processor that found the minimum cut bisection communicates its result to the root of its group (`myrank == 0`).
    *   The roots of these groups (where `myrank == 0`) then use another MPI communicator (`labelcomm`) and `MPI_Reduce` (MPI_SUM) to combine their separator results into the global `part` array for `agraph`. (Note: Summing labels might be a way to ensure all parts are covered, assuming non-overlap from `KeepPart` logic).
    *   The final global `part` array (representing the multisection for the entire assembled graph) is scattered from PE 0 (of `ctrl->comm`) to all PEs, updating their local `graph->where` arrays.
*   **`AssembleMultisectedGraph(ctrl_t *ctrl, graph_t *graph)`**:
    *   Gathers the entire distributed graph (`graph`) onto every processor (similar to `AssembleAdaptiveGraph` but tailored for multisection, specifically handling single-constraint vertex weights and initial `where` values).
    *   Each processor sends its local graph data (CSR structure, `vwgt[i]`, `where[i]`) to all other processors using `gkMPI_Allgatherv`.
    *   Each processor reconstructs the full graph (`agraph`) from the received data.
    *   `agraph->label` stores original global vertex indices. `agraph->where` stores initial partition assignments from input `graph->where`.

## Important Variables/Constants

*   **`agraph` (graph_t)**: The assembled graph structure containing the entire graph data, replicated on processors or subgroups.
*   **`part` (idx_t array)**: Used to store the computed global multisection partition for `agraph`.
*   **`label` (idx_t array in `agraph`)**: Stores the original global indices of vertices in `agraph`.
*   **`ctrl->nparts`**: On input, this usually indicates the number of desired partitions *after* the multisection (e.g., if `nparts=2`, it's a bisection; if `nparts=4`, it means an existing 2-way partition is further bisected). The output `where` array uses labels up to `ctrl->nparts + 2*(mypart_max)`.
*   **`METIS_OPTION_NSEPS`**: Option for `METIS_ComputeVertexSeparator` specifying how many separators to try.
*   **`METIS_OPTION_UFACTOR`**: Imbalance factor for METIS.
*   MPI Communicators: `newcomm` (for PE groups working on one sub-bisection), `labelcomm` (for PEs with `myrank==0` in their `newcomm` to combine results).

## Usage Examples

```
N/A
```
This is an internal ParMETIS function, typically called during multilevel partitioning (e.g., by `MultilevelBisect` or similar functions in `kmetis.c` or `ometis.c`) to find separators on the coarsest graph.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides `ctrl_t`, `graph_t` definitions, MPI wrappers, memory utilities.
    *   `graph.c`: `AssembleMultisectedGraph` uses `CreateGraph`, `imalloc`, `iincset`, `MAKECSR`. `InitMultisection` uses `AssembleMultisectedGraph`, `FreeGraph`.
    *   `serial.c`: `KeepPart` is used to isolate parts of the assembled graph.
    *   `metis.h` (external METIS library): `METIS_SetDefaultOptions`, `METIS_ComputeVertexSeparator`.
*   **External Libraries**:
    *   `MPI (Message Passing Interface)`: Used for communication (`gkMPI_Comm_split`, `gkMPI_Comm_rank`, `gkMPI_Allreduce`, `gkMPI_Send`, `gkMPI_Recv`, `gkMPI_Reduce`, `gkMPI_Scatterv`, `gkMPI_Comm_free`).
    *   `METIS`: The serial METIS library is used to compute vertex separators on subgraphs.
*   **Other Interactions**:
    *   The input `graph->where` array can specify an existing partition that needs to be further multisectioned.
    *   The output `graph->where` array is encoded such that partitions `0` to `nparts-1` are the sections, and partitions from `nparts` onwards represent the separators themselves (with a formula to link separator parts to the original sections they divide).
    *   This function is a key step in parallel multilevel k-way partitioning or ordering algorithms that rely on recursive bisection/multisection.

```
