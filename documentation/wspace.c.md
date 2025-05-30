# libparmetis/wspace.c

## Overview

This file manages workspace memory allocation and a specialized memory pool for `cnbr_t` structures (coarse neighbor information used in refinement) within ParMETIS. It utilizes GKlib's `mcore` functionality for efficient, stack-like workspace allocation, allowing for large temporary arrays to be allocated and freed in blocks. The `cnbr_t` pool is a dynamically growing array to store neighbor information during graph refinement.

## Key Components

### Functions/Classes/Modules

*   **`AllocateWSpace(ctrl_t *ctrl, size_t nwords)`**:
    *   Initializes the main workspace for ParMETIS.
    *   Creates a GKlib memory core (`ctrl->mcore`) of size `nwords * sizeof(idx_t)`. This core is used for most temporary allocations via `wspacemalloc` and its typed variants.
*   **`AllocateRefinementWorkSpace(ctrl_t *ctrl, idx_t nbrpoolsize)`**:
    *   Allocates and initializes a specialized memory pool (`ctrl->cnbrpool`) for storing `cnbr_t` structures. These structures hold information about neighboring partitions during refinement.
    *   `ctrl->nbrpoolsize`: Initial size of the pool.
    *   `ctrl->nbrpoolcpos`: Current position (next free entry).
    *   `ctrl->nbrpoolreallocs`: Counter for reallocations.
*   **`FreeWSpace(ctrl_t *ctrl)`**:
    *   Deallocates the main workspace memory core (`ctrl->mcore`).
    *   Prints statistics about the `cnbrpool` usage (size, current position, reallocations) if `DBG_INFO` is enabled.
    *   Frees the `ctrl->cnbrpool`.
*   **`wspacemalloc(ctrl_t *ctrl, size_t nbytes)`**:
    *   Allocates `nbytes` of memory from the GKlib memory core (`ctrl->mcore`). This is a stack-like allocation.
*   **`iwspacemalloc(ctrl_t *ctrl, size_t n)`**:
    *   Typed wrapper for `wspacemalloc`, allocating space for `n` `idx_t` elements.
*   **`rwspacemalloc(ctrl_t *ctrl, size_t n)`**:
    *   Typed wrapper for `wspacemalloc`, allocating space for `n` `real_t` elements.
*   **`ikvwspacemalloc(ctrl_t *ctrl, size_t n)`**:
    *   Typed wrapper for `wspacemalloc`, allocating space for `n` `ikv_t` elements.
*   **`rkvwspacemalloc(ctrl_t *ctrl, size_t n)`**:
    *   Typed wrapper for `wspacemalloc`, allocating space for `n` `rkv_t` elements.
*   **`cnbrpoolReset(ctrl_t *ctrl)`**:
    *   Resets the current position pointer (`ctrl->nbrpoolcpos`) of the `cnbrpool` to 0, effectively clearing the pool for reuse without deallocating memory.
*   **`cnbrpoolGetNext(ctrl_t *ctrl, idx_t nnbrs)`**:
    *   Gets a block of `nnbrs` `cnbr_t` entries from the `cnbrpool`.
    *   `nnbrs` is capped by `ctrl->nparts`.
    *   If the pool is too small, it reallocates `ctrl->cnbrpool` (increasing its size by `max(10*nnbrs, current_size/2)`).
    *   Returns the starting index of the allocated block in the pool.

## Important Variables/Constants

*   **`ctrl->mcore` (gk_mcore_t *)**: Pointer to the GKlib memory core used for general workspace allocations.
*   **`ctrl->nbrpoolsize` (size_t)**: Current allocated size (number of `cnbr_t` elements) of the neighbor pool.
*   **`ctrl->nbrpoolcpos` (size_t)**: Index of the next available free entry in `cnbrpool`.
*   **`ctrl->nbrpoolreallocs` (size_t)**: Number of times `cnbrpool` has been reallocated.
*   **`ctrl->cnbrpool` (cnbr_t *)**: The actual memory pool for storing `cnbr_t` structures.

## Usage Examples

```c
// General workspace allocation:
// WCOREPUSH; // Start a new scope (macro from macros.h)
// idx_t *my_temp_array = iwspacemalloc(ctrl, 1000);
// real_t *another_temp = rwspacemalloc(ctrl, 500);
// // ... use arrays ...
// WCOREPOP; // Frees my_temp_array and another_temp (and anything else allocated via wspacemalloc in this scope)

// Coarse neighbor pool usage (typically in refinement functions):
// // At the start of a refinement phase:
// cnbrpoolReset(ctrl);
//
// // When processing a vertex 'v' needing 'num_neighbors' entries:
// idx_t ckr_inbr = cnbrpoolGetNext(ctrl, num_neighbors);
// cnbr_t *mynbrs = ctrl->cnbrpool + ckr_inbr;
// // ... populate mynbrs[0]...mynbrs[num_neighbors-1] ...
```

## Dependencies and Interactions

*   **Internal Dependencies**:
    *   `parmetislib.h`: Provides `ctrl_t`, `cnbr_t`, `ikv_t`, `rkv_t`, `idx_t`, `real_t` types, and GKlib types/functions (`gk_mcore_t`, `gk_mcoreCreate`, `gk_mcoreDestroy`, `gk_mcoreMalloc`, `gk_malloc`, `gk_realloc`, `gk_free`, `gk_min`, `gk_max`).
    *   `DBG_INFO` (from `defs.h` via `parmetislib.h`): Controls printing of `cnbrpool` statistics.
*   **External Libraries**:
    *   GKlib (embedded): The memory core (`mcore`) functionality and standard memory allocation wrappers (`gk_malloc`, etc.) are from GKlib.
*   **Other Interactions**:
    *   `AllocateWSpace` is typically called once at the beginning of a ParMETIS API function to set up the main temporary memory region.
    *   `AllocateRefinementWorkSpace` is called before refinement phases that require `ckrinfo_t` and `cnbr_t` structures (e.g., `ComputePartitionParams`, `KWayFM`).
    *   `wspacemalloc` and its typed variants are used throughout ParMETIS for allocating temporary arrays needed within a specific scope, often bracketed by `WCOREPUSH` and `WCOREPOP` (macros from `macros.h`) for automatic cleanup.
    *   The `cnbrpool` provides a dynamic array for storing adjacency information with respect to partitions, which is heavily used during k-way refinement.

```
