# libparmetis/graph.c

## Overview

This file contains routines for managing the `graph_t` data structure in ParMETIS. This includes functions for creating, initializing, and freeing graph structures, as well as setting up various graph attributes like normalized vertex weights (`nvwgt`). It also includes experimental functionality for writing and reading graph data to/from disk to potentially save memory for very large graphs during multilevel processing.

## Key Components

### Functions/Classes/Modules

*   **`SetupGraph(ctrl_t *ctrl, idx_t ncon, idx_t *vtxdist, idx_t *xadj, idx_t *vwgt, idx_t *vsize, idx_t *adjncy, idx_t *adjwgt, idx_t wgtflag)`**:
    *   Creates and initializes a `graph_t` structure from user-provided raw graph data.
    *   Sets basic graph properties: `nvtxs`, `nedges`, `ncon`, CSR arrays (`xadj`, `adjncy`), `vtxdist`.
    *   Allocates and initializes vertex weights (`vwgt`) and edge weights (`adjwgt`) if not provided by the user (based on `wgtflag`).
    *   Allocates `vsize` (vertex sizes) and `home` arrays if the operation type is adaptive or refinement.
    *   Calculates `ctrl->edge_size_ratio`.
    *   Computes inverse total vertex weights (`SetupCtrl_invtvwgts`).
    *   Computes normalized vertex weights (`SetupGraph_nvwgts`).
*   **`SetupGraph_nvwgts(ctrl_t *ctrl, graph_t *graph)`**:
    *   Computes the normalized vertex weights (`graph->nvwgt`) for each constraint.
    *   `nvwgt[i*ncon+j] = graph->vwgt[i*ncon+j] * ctrl->invtvwgts[j]`.
*   **`CreateGraph(void)`**:
    *   Allocates memory for a `graph_t` structure and initializes its fields to default null/invalid values using `InitGraph`.
*   **`InitGraph(graph_t *graph)`**:
    *   Sets all fields of a `graph_t` structure to zero or default invalid values (e.g., -1, NULL).
    *   Initializes `free_*` flags to 1 (indicating ParMETIS should free these arrays).
*   **`FreeGraph(graph_t *graph)`**:
    *   Frees all memory associated with a `graph_t` structure, including base graph arrays (xadj, vwgt, etc.) and other auxiliary arrays (match, cmap, where, etc.). It calls `FreeNonGraphFields`.
*   **`FreeNonGraphFields(graph_t *graph)`**:
    *   Frees most auxiliary arrays within the `graph_t` structure, excluding the primary graph data (xadj, adjncy, vwgt, adjwgt, vsize, vtxdist, nvwgt, home).
*   **`FreeNonGraphNonSetupFields(graph_t *graph)`**:
    *   Frees auxiliary arrays similar to `FreeNonGraphFields` but also keeps communication setup related fields.
*   **`FreeCommSetupFields(graph_t *graph)`**:
    *   Specifically frees arrays related to communication setup (lperm, peind, sendptr, etc.).
*   **`FreeInitialGraphAndRemap(graph_t *graph)`**:
    *   Frees a graph structure after partitioning, including performing a local-to-global remapping of `adjncy` if `graph->imap` exists.
    *   Selectively frees `vwgt`, `adjwgt`, `vsize` based on their `free_*` flags.
*   **`graph_WriteToDisk(ctrl_t *ctrl, graph_t *graph)`**:
    *   (Experimental) If `ctrl->ondisk` is enabled and graph size exceeds a threshold, writes key graph arrays (xadj, vwgt, nvwgt, adjncy, adjwgt, vsize, home) to a binary file.
    *   Frees these arrays from memory after writing. Sets `graph->ondisk = 1`.
*   **`graph_ReadFromDisk(ctrl_t *ctrl, graph_t *graph)`**:
    *   (Experimental) If `graph->ondisk` is set, reads the graph data previously written by `graph_WriteToDisk` back into memory.
    *   Deletes the disk file after reading.

## Important Variables/Constants

*   **`graph_t` (struct)**: The central data structure for representing distributed graphs. Key members include:
    *   `nvtxs`, `nedges`, `ncon`: Local number of vertices, edges, constraints.
    *   `gnvtxs`: Global number of vertices.
    *   `xadj`, `adjncy`, `vwgt`, `adjwgt`, `vsize`: CSR graph structure and weights.
    *   `nvwgt`: Normalized vertex weights.
    *   `vtxdist`: Distribution of vertices to PEs.
    *   `home`: Original partition/PE of a vertex (for adaptive/refinement).
    *   `where`: Current partition assignment.
    *   Communication fields: `peind`, `sendptr`, `sendind`, `recvptr`, `recvind`, `imap`, `pexadj`, etc. (managed by `comm.c`).
    *   Coarsening fields: `match`, `cmap`.
    *   `coarser`, `finer`: Pointers to graphs at different levels in the multilevel hierarchy.
    *   `free_xadj`, `free_vwgt`, etc.: Flags indicating if ParMETIS allocated and is responsible for freeing these arrays.
    *   `ondisk`, `gID`: Fields for managing on-disk storage of graph data.
*   **`ctrl_t` (struct)**: Control structure, used for `optype`, `mype`, `npes`, `ondisk` flag, `invtvwgts`.

## Usage Examples

```
N/A
```
These are internal ParMETIS functions.
*   `SetupGraph` is called at the beginning of most ParMETIS API functions to convert user input into the internal `graph_t` format.
*   `CreateGraph`/`InitGraph` are used when building new graph structures (e.g., coarser graphs).
*   The `Free*` functions are used for memory management throughout the lifecycle of graph structures, especially in the multilevel coarsening/refinement process.
*   `graph_WriteToDisk` and `graph_ReadFromDisk` might be called during multilevel algorithms if the `ondisk` option is active to reduce memory footprint.

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides `ctrl_t`, `graph_t` definitions, `idx_t`, `real_t` types, memory management (`gk_malloc`, `ismalloc`, `rmalloc`, `gk_free`, `memset`), and GKlib utilities (like `isum` used via `ctrl.c`).
    *   `ctrl.c`: `SetupCtrl_invtvwgts` is called by `SetupGraph`.
    *   `comm.c`: The communication fields initialized as `NULL` here are populated by `CommSetup` from `comm.c`.
    *   The `graph_t` structure is fundamental and is passed to nearly all ParMETIS functions.
*   **External Libraries**:
    *   Standard C I/O (`stdio.h`): Used for `fopen`, `fwrite`, `fread`, `fclose` in `graph_WriteToDisk`/`graph_ReadFromDisk`.
    *   Standard C library (`string.h` for `memset`).
*   **Other Interactions**:
    *   The `free_*` flags in `graph_t` determine whether ParMETIS reclaims memory for graph arrays or assumes the user will manage them (if they were passed in and `wgtflag` indicated user-provided arrays).
    *   The on-disk functionality creates temporary files named `parmetis<PE>.<PID>.<gID>`.

```
